# macOS xpc\_connection\_get\_audit\_token Attack

<details>

<summary><strong>Learn AWS hacking from zero to hero with</strong> <a href="https://training.hacktricks.xyz/courses/arte"><strong>htARTE (HackTricks AWS Red Team Expert)</strong></a><strong>!</strong></summary>

Other ways to support HackTricks:

* If you want to see your **company advertised in HackTricks** or **download HackTricks in PDF** Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!
* Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)
* Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)
* **Join the** 💬 [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** 🐦 [**@carlospolopm**](https://twitter.com/carlospolopm)**.**
* **Share your hacking tricks by submitting PRs to the** [**HackTricks**](https://github.com/carlospolop/hacktricks) and [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repos.

</details>

## Mach Messages Basic Info

If you don't know what Mach Messages are start checking this page:

{% content-ref url="../../../../mac-os-architecture/macos-ipc-inter-process-communication/" %}
[macos-ipc-inter-process-communication](../../../../mac-os-architecture/macos-ipc-inter-process-communication/)
{% endcontent-ref %}

For the moment remember that:\
Mach messages are sent over a _mach port_, which is a **single receiver, multiple sender communication** channel built into the mach kernel. **Multiple processes can send messages** to a mach port, but at any point **only a single process can read from it**. Just like file descriptors and sockets, mach ports are allocated and managed by the kernel and processes only see an integer, which they can use to indicate to the kernel which of their mach ports they want to use.

## XPC Connection

If you don't know how a XPC connection is established check:

{% content-ref url="../" %}
[..](../)
{% endcontent-ref %}

## Vuln Summary

What is interesting for you to know is that **XPC’s abstraction is a one-to-one connection**, but it is based on top of a technology which **can have multiple senders, so:**

* Mach ports are single receiver, **multiple sender**.
* An XPC connection’s audit token is the audit token of **copied from the most recently received message**.
* Obtaining the **audit token** of an XPC connection is critical to many **security checks**.

Although the previous situation sounds promising there are some scenarios where this is not going to cause problems:

* Audit tokens are often used for an authorization check to decide whether to accept a connection. As this happens using a message to the service port, there is **no connection established yet**. More messages on this port will just be handled as additional connection requests. So any **checks before accepting a connection are not vulnerable** (this also means that within `-listener:shouldAcceptNewConnection:` the audit token is safe). We are therefore **looking for XPC connections that verify specific actions**.
* XPC event handlers are handled synchronously. This means that the event handler for one message must be completed before calling it for the next one, even on concurrent dispatch queues. So inside an **XPC event handler the audit token can not be overwritten** by other normal (non-reply!) messages.

This gave us the idea for two different methods this may be possible:

1. Variant1:
   * **Exploit** **connects** to service **A** and service **B**
     * Service **B** can call a **privileged functionality** in service A that the user cannot
   * Service **A** calls **`xpc_connection_get_audit_token`** while _**not**_ inside the **event handler** for a connection in a **`dispatch_async`**.
     * So a **different** message could **overwrite the Audit Token** because it's being dispatched asynchronously outside of the event handler.
   * The exploit passes to **service B the SEND right to service A**.
     * So svc **B** will be actually **sending** the **messages** to service **A**.
   * The **exploit** tries to **call** the **privileged action.** In a RC svc **A** **checks** the authorization of this **action** while **svc B overwrote the Audit token** (giving the exploit access to call the privileged action).
2. Variant 2:
   * Service **B** can call a **privileged functionality** in service A that the user cannot
   * Exploit connects with **service A** which **sends** the exploit a **message expecting a response** in a specific **replay** **port**.
   * Exploit sends **service** B a message passing **that reply port**.
   * When service **B replies**, it s**ends the message to service A**, **while** the **exploit** sends a different **message to service A** trying to **reach a privileged functionality** and expecting that the reply from service B will overwrite the Audit token in the perfect moment (Race Condition).

## Variant 1: calling xpc\_connection\_get\_audit\_token outside of an event handler <a href="#variant-1-calling-xpc_connection_get_audit_token-outside-of-an-event-handler" id="variant-1-calling-xpc_connection_get_audit_token-outside-of-an-event-handler"></a>

Scenario:

* Two mach **services **`A`** and **`B`** that we can both connect to (based on the sandbox profile and the authorization checks before accepting the connection).
* _**A**_ must have an **authorization check** for a specific action that **`B`** can pass (but our app can’t).
  * For example, if B has some **entitlements** or is running as **root**, it might allow him to ask A to perform a privileged action.
* For this authorization check, **`A`** obtains the audit token asynchronously, for example by calling `xpc_connection_get_audit_token` from **`dispatch_async`**.

{% hint style="danger" %}
In this case an attacker could trigger a **Race Condition** making a **exploit** that **asks A to perform an action** several times while making **B send messages to `A`**. When the RC is **successful**, the **audit token** of **B** will be copied in memory **while** the request of our **exploit** is being **handled** by A, giving it **access to the privilege action only B could request**.
{% endhint %}

This happened with **`A`** as `smd` and **`B`** as `diagnosticd`. The function [`SMJobBless`](https://developer.apple.com/documentation/servicemanagement/1431078-smjobbless?language=objc) from smb an be used to install a new privileged helper toot (as **root**). If a **process running as root contact** **smd**, no other checks will be performed.

Therefore, the service **B** is **`diagnosticd`** because it runs as **root** and can be used to **monitor** a process, so once monitoring has started, it will **send multiple messages per second.**

To perform the attack:

1. We establish our **connection** to **`smd`** by following the normal XPC protocol.
2. Then, we establish a **connection** to **`diagnosticd`**, but instead of generating two new mach ports and sending those, we replace the client port send right with a copy of the **send right we have for the connection to `smd`**.
3. What this means is that we can send XPC messages to `diagnosticd`, but any **messages `diagnosticd` sends go to `smd`**.
   * For `smd`, both our and `diagnosticd`’s messages appear arrive on the same connection.

<figure><img src="../../../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1).png" alt="" width="563"><figcaption></figcaption></figure>

4. We ask **`diagnosticd`** to **start monitoring** our (or any active) process and we **spam routine 1004 messages to `smd`** (to install a privileged tool).
5. This creates a race condition that needs to hit a very specific window in `handle_bless`. We need the call to `xpc_connection_get_pid` to return the PID of our own process, as the privileged helper tool is in our app bundle. However, the call to `xpc_connection_get_audit_token` inside the `connection_is_authorized` function must use the audit token of `diganosticd`.

## Variant 2: reply forwarding

As mentioned before, the handler for events on an XPC connection is never executed multiple times concurrently. However, **XPC reply**** messages are handled differently. Two functions exist for sending a message that expects a reply:

* `void xpc_connection_send_message_with_reply(xpc_connection_t connection, xpc_object_t message, dispatch_queue_t replyq, xpc_handler_t handler)`, in which case the XPC message is received and parsed on the specified queue.
* `xpc_object_t xpc_connection_send_message_with_reply_sync(xpc_connection_t connection, xpc_object_t message)`, in which case the XPC message is received and parsed on the current dispatch queue.

Therefore, **XPC reply packets may be parsed while an XPC event handler is executing**. While `_xpc_connection_set_creds` does use locking, this only prevents partial overwriting of the audit token, it does not lock the entire connection object, making it possible to **replace the audit token in between the parsing** of a packet and the execution of its event handler.

For this scenario we would need:

* As before, two mach services **`A`** and **`B`** that we can both connect to.
* Again, **`A`** must have an authorization check for a specific action that **`B`** can pass (but our app can’t).
* **`A`** sends us a message that expects a reply.
* We can send a message to **`B`** that it will reply to.

We wait for **`A`** to send us a message that expects a reply (1), instead of replying we take the reply port and use it for a message we send to **`B`** (2). Then, we send a message that uses the forbidden action and we hope that it arrives concurrently with the reply from **`B`** (3).

<figure><img src="../../../../../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1).png" alt="" width="563"><figcaption></figcaption></figure>

## Discovery Problems

We spent a long time trying to find other instances, but the conditions made it difficult to search for either statically or dynamically. To search for asynchronous calls to `xpc_connection_get_audit_token`, we used Frida to hook on this function to check if the backtrace includes `_xpc_connection_mach_event` (which means it’s not called from an event handler). But this only finds calls in the process we have currently hooked and from the actions that are actively used. Analysing all reachable mach services in IDA/Ghidra was very time intensive, especially when calls involved the dyld shared cache. We tried scripting this to look for calls to `xpc_connection_get_audit_token` reachable from a block submitted using `dispatch_async`, but parsing blocks and calls passing into the dyld shared cache made this difficult too. After spending a while on this, we decided it would be better to submit what we had.

## The fix <a href="#the-fix" id="the-fix"></a>

In the end, we reported the general issue and the specific issue in `smd`. Apple fixed it only in `smd` by replacing the call to `xpc_connection_get_audit_token` with `xpc_dictionary_get_audit_token`.

The function `xpc_dictionary_get_audit_token` copies the audit token from the mach message on which this XPC message was received, meaning it is not vulnerable. However, just like `xpc_dictionary_get_audit_token`, this is not part of the public API. For the higher level `NSXPCConnection` API, no clear method exists to get the audit token of the current message, as this abstracts away all messages into method calls.

It is unclear to us why Apple didn’t apply a more general fix, for example dropping messages that don’t match the saved audit token of the connection. There may be scenarios where the audit token of a process legitimately changes but the connection should stay open (for example, calling `setuid` changes the UID field), but changes like a different PID or PID version are unlikely to be intended.

In any case, this issue still remains with iOS 17 and macOS 14, so if you want to go and look for it, good luck!

# References
* For further information check the original post: [https://sector7.computest.nl/post/2023-10-xpc-audit-token-spoofing/](https://sector7.computest.nl/post/2023-10-xpc-audit-token-spoofing/)

<details>

<summary><strong>Learn AWS hacking from zero to hero with</strong> <a href="https://training.hacktricks.xyz/courses/arte"><strong>htARTE (HackTricks AWS Red Team Expert)</strong></a><strong>!</strong></summary>

Other ways to support HackTricks:

* If you want to see your **company advertised in HackTricks** or **download HackTricks in PDF** Check the [**SUBSCRIPTION PLANS**](https://github.com/sponsors/carlospolop)!
* Get the [**official PEASS & HackTricks swag**](https://peass.creator-spring.com)
* Discover [**The PEASS Family**](https://opensea.io/collection/the-peass-family), our collection of exclusive [**NFTs**](https://opensea.io/collection/the-peass-family)
* **Join the** 💬 [**Discord group**](https://discord.gg/hRep4RUj7f) or the [**telegram group**](https://t.me/peass) or **follow** me on **Twitter** 🐦 [**@carlospolopm**](https://twitter.com/carlospolopm)**.**
* **Share your hacking tricks by submitting PRs to the** [**HackTricks**](https://github.com/carlospolop/hacktricks) and [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repos.

</details>
