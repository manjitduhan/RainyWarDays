# 🧠 An IPsec Tunnel That Came Up — But No Traffic Was Flowing
_A Debugging Story Across strongSwan, PF_KEY, and the Kernel_

---

## 🪧 Introduction
In production, everything looked green. All peers were connected, IPsec tunnels were established, and monitoring tools reported no errors.
Then came a compliance flag: “Some devices are still using SHA1 — migrate to SHA256 immediately.”

As security owners, we knew this wasn’t optional. We prioritized the change, updated the configuration on both initiator and responder to use HMAC-SHA256, and rolled it out.

And then — chaos.

* Client-side applications started failing silently.
* Server reported client as disconnected.
* No traffic appeared at the server for that client, yet tunnels showed as UP.
* IPsec Keepalive packets flowed.
* Outbound packet counters on the client kept increasing.
* But Inbount packet counter on server for that tunnel was 0.

It was a classic case of “the tunnel is up, but nothing works.”


### 🧪 The Initial Hypothesis
We took no chances — we immediately reverted to SHA1. The clients recovered. That was our smoking gun.

To isolate the issue, we spun up a test environment with SHA256 enabled.
Same failure. This replication gave us control — and hope.

### 🛠️ Diving Into the Stack
We pulled out our usual toolkit:

- strongSwan logs? Clean.
- ipsec statusall? Tunnel up.
- tcpdump? Encrypted ESP Packets are coming at the server.
- xfrm statistics? Aha!
Packets were being silently dropped due to integrity check failures.

This pointed to an ICV (Integrity Check Value) mismatch. But why?

🧬 Anatomy of an Invisible Failure
As we dug deeper, we discovered that the kernel was using a truncated 96-bit ICV for HMAC-SHA256 — not the expected 128 bits.

We confirmed this via the kernel source:

📂 [Linux - xfrm_algo.c](https://github.com/torvalds/linux/blob/master/net/xfrm/xfrm_algo.c#L235)
```
.name = "hmac(sha256)",
.compat = "sha256",
.uinfo = {
  .auth = {
    .icv_truncbits = 96, //<---This is something fishy here
    .icv_fullbits = 256,
  }
},
.pfkey_supported = 1,
```
This meant:

The Linux kernel silently defaulted to 96-bit ICVs

Our remote peer was using 128-bit ICVs (per [RFC 4868 §2.3](https://www.rfc-editor.org/rfc/rfc4868#section-2.3))

Result: packets failed integrity checks and were dropped — without any warning in the logs

### 📌 RFC 4868 recommends ICV length = L/2, where L is the hash output length (256 bits → 128 bits).

So… why didn’t the kernel match this?

## 🧩 The Plugin That Made All the Difference
Our focus shifted from the kernel to how strongSwan communicates with the kernel.

We explored two kernel interface plugins:

- kernel-pfkey
- kernel-netlink

On inspecting strongSwan’s PF_KEY plugin, we noticed it does not set the ICV truncation length explicitly.
So the kernel defaults to 96 bits.

In contrast, the kernel-netlink plugin does the right thing:

📂 [strongSwan - kernel_netlink_ipsec.c](https://github.com/strongswan/strongswan/blob/master/src/libcharon/plugins/kernel_netlink/kernel_netlink_ipsec.c#L2008C1-L2026C4)
```
if (trunc_len)
{
	struct xfrm_algo_auth* algo;

	/* the kernel uses SHA256 with 96 bit truncation by default,
	 * use specified truncation size supported by newer kernels.
	 * also use this for untruncated MD5, SHA1 and SHA2. */
	algo = netlink_reserve(hdr, sizeof(request), XFRMA_ALG_AUTH_TRUNC,
						   sizeof(*algo) + data->int_key.len);
	if (!algo)
	{
		goto failed;
	}
	algo->alg_key_len = data->int_key.len * 8;
	algo->alg_trunc_len = trunc_len;
	strncpy(algo->alg_name, alg_name, sizeof(algo->alg_name)-1);
	algo->alg_name[sizeof(algo->alg_name)-1] = '\0';
	memcpy(algo->alg_key, data->int_key.ptr, data->int_key.len);
}
```

Here, the plugin explicitly sets the truncation length — 128 bits in our case — aligning perfectly with the peer.

Now the picture was complete:

Our strongSwan installation had both pfkey and netlink plugins installed.

PF_KEY took precedence, silently skipping ICV length specification.

The kernel defaulted to 96 bits, causing all incoming packets to fail integrity checks.

## 🧹 The Fix
We disabled the kernel-pfkey plugin and retained only kernel-netlink.

✅ ICV properly negotiated

✅ Packets decrypted successfully

✅ Application recovered

✅ RFC compliance ensured

## 🎯 Key Learnings for Security Architects
Tunnel status != Traffic flow
Always inspect xfrm statistics and kernel drop counters.

Defaults can bite
The kernel’s default truncation for HMAC-SHA256 is 96 bits, not RFC-recommended 128.

Not all strongSwan plugins are equal
kernel-pfkey lacks full control over modern IPsec parameters. Prefer kernel-netlink.

Logs may not save you
Neither strongSwan nor the system kernel logs flagged this integrity mismatch.

RFCs matter
Compliance to cryptographic recommendations is subtle but essential — always double-check specs like RFC 4868.

## 🧠 Final Thoughts
This issue pushed us through every layer of the IPsec stack — from application outage to RFCs, from kernel code to strongSwan plugin internals.
It’s a reminder that cryptographic configurations are never “just config.” They are contracts — and every byte matters.

As architects, it’s our job to understand not just how things are wired — but how the defaults behave when left unchecked.

This wasn’t just a bug.
It was a blind spot in design — and we fixed it with understanding, not guesswork.
