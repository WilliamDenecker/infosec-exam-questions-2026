# Question 51

**Does authenticated Diffie-Hellman (DH) guarantee forward secrecy? What happens if one or both of the fixed DH keys is compromised? What is the impact on future (i.e. after the keys have been compromised) communications? What is the impact on past (i.e. before the keys have been compromised) communications?**

## Answer

### What Is "Authenticated DH"? (ch2.2.4 p.11–15)

The slides describe **authenticated DH** as a variant combining fixed (long-term) DH key pairs for authentication with session-specific ephemeral parameters for key derivation (ch2.2.4 p.11–12):

- **Fixed parameters**: Alice holds $(X_A, Y_A = a^{X_A} \bmod q)$; Bob holds $(X_B, Y_B = a^{X_B} \bmod q)$
- **Long-term shared secret**: $K_{AB} = a^{X_A X_B} \bmod q$ (derivable by either party from their fixed private key and the other's public key)
- **Session parameters**: Alice generates ephemeral $\alpha$ and sends $a^{\alpha \cdot K_{AB}} \bmod q$; Bob generates ephemeral $\beta$ and sends $a^{\beta \cdot K_{AB}} \bmod q$
- **Session key**: $K_S = a^{\alpha\beta} \bmod q$ (derivable by Alice as $(a^{\beta K_{AB}})^{K_{AB}^{-1}\alpha}$ and by Bob symmetrically — only possible if one knows $K_{AB}$)

Authentication is provided because only a party that knows $K_{AB}$ (derived from the fixed DH key pair) can compute $K_S$ from the observed session values.

---

### Does Authenticated DH Guarantee Forward Secrecy? (ch2.2.4 p.11–13)

**Partial answer — depends on what is compromised and what is kept secret**.

**With properly managed ephemeral parameters**: the session key $K_S = a^{\alpha\beta} \bmod q$ does not depend directly on $K_{AB}$ as a value — $K_{AB}$ enters only as a multiplier of the exponents. If $\alpha$ and $\beta$ are discarded after the session, recovering $K_S$ from the recorded traffic requires solving a Diffie-Hellman problem (finding $a^{\alpha\beta}$ from $a^{\alpha K_{AB}}$ and $a^{\beta K_{AB}}$), even for an attacker who later learns $K_{AB}$.

**However**: if the fixed DH private keys $X_A$ or $X_B$ are compromised, the attacker can compute $K_{AB} = Y_B^{X_A} = a^{X_A X_B}$. With $K_{AB}$ known:
- For **future sessions**: the attacker can impersonate either party (compute session values that the other party will accept), breaking authentication entirely (ch2.2.4 p.13)
- For **past sessions**: if $\alpha$ and $\beta$ were properly discarded, the attacker still faces a DH problem to recover $K_S$ from the recorded $a^{\alpha K_{AB}}$ and $a^{\beta K_{AB}}$ — but if they can solve DLP (which Shor's algorithm enables, ch2.2 PQCrypto p.12), they can recover $\alpha K_{AB}$ from $a^{\alpha K_{AB}}$, divide by $K_{AB}$ to get $\alpha$, and then compute $K_S = (a^{\beta K_{AB}})^{\alpha / K_{AB}} = a^{\alpha\beta}$

---

### Scenario Analysis

#### If One Fixed Key Is Compromised

Say Alice's private key $X_A$ is compromised:
- Attacker computes $K_{AB} = Y_B^{X_A} \bmod q$
- **Future sessions**: attacker can impersonate Alice — authentication is broken; MITM attacks possible (ch2.2.4 p.13)
- **Future sessions**: attacker can also verify $K_S$ derivations, fully compromising new sessions
- **Past sessions** (where $\alpha, \beta$ were discarded): if the attacker cannot solve DLP (classical adversary), $K_S$ from past sessions remains computationally secure. But if the attacker has a quantum computer (Shor's), they can recover $\alpha$ and $\beta$ from the recorded values, computing $K_S$ retroactively
- **Conclusion**: past sessions are conditionally secure (against classical attacker with discarded $\alpha$, $\beta$); future sessions are fully compromised

#### If Both Fixed Keys Are Compromised

- Attacker knows $X_A$ and $X_B$ (or equivalently, $K_{AB}$ directly)
- **Future sessions**: fully compromised — attacker can impersonate both parties and compute all future $K_S$ values
- **Past sessions**: same analysis as above — classically secure if $\alpha$, $\beta$ were discarded and DLP is hard; broken retroactively with a quantum computer

#### If Only Ephemeral Parameters Are Compromised (Not Fixed Keys)

- If $\alpha$ from one past session is exposed: the attacker can compute $K_S = (a^{\beta K_{AB}})^{K_{AB}^{-1}\alpha}$ — only that session is compromised, not others (each session uses independent $\alpha$, $\beta$)
- Fixed keys and all other sessions remain secure — this is the forward-secrecy property of per-session ephemeral keys

---

### Comparison: Fixed DH vs Ephemeral DH Authentication (ch2.2.4 p.11–15)

| Aspect | Fixed DH (authenticated DH as in slides) | Ephemeral DH with signature authentication |
|---|---|---|
| Authentication mechanism | Knowledge of $K_{AB}$ (from fixed key pair) | Digital signature with long-term RSA/ECDSA key |
| Session key | $K_S = a^{\alpha\beta}$ — ephemeral | $K_S = a^{\alpha\beta}$ — ephemeral |
| Compromise of auth key | $K_{AB}$ leaked → past sessions potentially recoverable with DLP | Signature key leaked → past sessions secure (signature doesn't affect $K_S$) |
| Forward secrecy for past sessions | Conditional (depends on DLP hardness) | Full (signature key not used in $K_S$ derivation) |
| Forward secrecy for future sessions | Broken | Broken (attacker can forge authentication) |

**Ephemeral DH** (DHE/ECDHE) with authentication via separate signatures (as used in TLS 1.3, ch3.6 p.7–8) provides **perfect forward secrecy** because the signature key is not used in the session key derivation at all — compromising the signature key allows future MITM attacks but cannot retroactively decrypt past sessions.

---

### Summary

- **Authenticated DH with fixed keys**: provides partial forward secrecy — past sessions are conditionally protected if ephemeral parameters ($\alpha$, $\beta$) are discarded, but rely on DLP hardness. Future sessions are fully compromised when fixed keys are exposed.
- **Past communications**: secure (classical adversary, discarded ephemerals); insecure if DLP can be solved (e.g., with a quantum computer)
- **Future communications**: authentication broken; full compromise when fixed keys are known
- **Best practice**: use ephemeral DH (DHE/ECDHE) with separate signature-based authentication to achieve true perfect forward secrecy, where compromise of the long-term signature key does not affect any past session key

### Sources

- IS_UG_2_2_4_SecM_KeyExch (p.11–12: authenticated DH — fixed key pairs, long-term shared secret $K_{AB}$, session parameters $\alpha$ and $\beta$, session key derivation; p.13: MITM attacks and when authentication breaks down; p.14–15: DoS considerations for DH variants)
- IS_UG_2_2_SecM-adv-PQCrypto (p.12: Shor's algorithm breaks discrete logarithm — retroactive session key recovery from fixed DH key compromise)
- IS_UG_3_6_Appl_TLS (p.7–8: TLS 1.3 ECDHE with signature authentication — true forward secrecy)

_Status: Complete_  
_Done by: William_
