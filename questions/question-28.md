# Question 28

We have seen in the course how the use of a *salt* in the encoded storage of a password could improve the security of the storage. The salt is typically unique for each user, but not secret (it is often stored in plain text).

An alternative technique is using both a *salt* and a *pepper* in the encoded storage. The pepper is identical for all users within the system but secret (only known to the application processing the passwords and not stored in plain text). This means that the password $P_i$ for some user $i$ (with salt $S_i$) will be stored as the encoded password $EP_i$:

$$EP_i = H(P_i\|S_i\|Pepper) \quad (Q28.5)$$

where $Pepper$ is the pepper used in this system, $H$ is the (one-way) encoding function, and $\|$ stands for the concatenation of data.

**Compare this use of a pepper + salt combination to the use of only a salt or only a pepper for secure password storage. What are the advantages/drawbacks of this pepper + salt combination?**

**Consider direct login attempts, dictionary attacks, and rainbow tables.**

## Answer

### Password Storage Baseline (ch3.2 p.11)

Without any protection, passwords are stored as $H(P_i)$. This allows precomputed lookup tables and dictionary attacks. The salt and pepper are countermeasures with different properties.

---

### Salt Only: $EP_i = H(P_i \| S_i)$ (ch3.2 p.11)

The salt $S_i$ is unique per user, stored in plaintext alongside the hash.

**Direct login**: attacker who obtains $EP_i$ and $S_i$ can still compute $H(P_{guess} \| S_i)$ for guesses. The salt provides no protection against per-user brute force — it only prevents cross-user attack (two users with the same password have different hashes).

**Dictionary attack**: without salt, a dictionary can be hashed once and compared against all users (any match reveals that user's password). With per-user salt, each user's entry requires its own dictionary computation — the attacker must rehash the entire dictionary for each user. This multiplies the attacker's workload by the number of users in the database — significant but not fundamental (if the dictionary is small, this is still feasible).

**Rainbow tables** (ch3.2 p.11): rainbow tables are precomputed hash chains for a specific hash function $H$. Without salt, a rainbow table can be used to look up any stolen hash instantly. With per-user salt, a rainbow table would need to be precomputed for every possible salt value — for a random 128-bit salt, this is computationally infeasible. **Salt completely defeats rainbow table attacks.**

**Weakness of salt only**: if the entire database (hashes + salts) is stolen, the attacker can dictionary-attack each account individually. The salt does not prevent this — it only prevents attacks that exploit multiple users at once.

---

### Pepper Only: $EP_i = H(P_i \| Pepper)$

The pepper is a single secret value, the same for all users.

**Direct login**: without the pepper, an attacker who steals the hash database cannot compute $H(P_{guess} \| Pepper)$ because Pepper is unknown. This is significant protection — it converts a pure one-way function into a keyed function (similar to HMAC) where the key is Pepper.

**Dictionary attack**: requires knowing Pepper. Without Pepper, even a brute-force dictionary attack on a single user is infeasible (each guess requires computing $H(P_{guess} \| Pepper)$, but Pepper is unknown). If Pepper is compromised (e.g., code repository leak), the protection is completely lost for all users simultaneously.

**Rainbow tables**: without Pepper, precomputed tables for $H(\cdot \| Pepper)$ are infeasible since Pepper is unknown. If Pepper is leaked, the attacker can build a rainbow table for the specific Pepper value and attack all users whose hashes are known.

**No per-user uniqueness**: two users with the same password have the same hash. An attacker who discovers one user's password by any means knows all other users with the same password.

**Critical weakness of pepper only**: if the pepper is compromised, all users are compromised simultaneously — there is no per-user protection. The pepper is a single point of failure.

---

### Salt + Pepper Combination: $EP_i = H(P_i \| S_i \| Pepper)$

**Direct login attack**:

An attacker attempting to log in as user $i$ must supply $P_{guess}$ and know $S_i$ (available from the database) and Pepper (secret). The server computes $H(P_{guess} \| S_i \| Pepper)$ and compares to $EP_i$. The server can detect failed attempts (brute force protection via rate limiting, ch3.7 p.85). An external attacker making direct login attempts is limited by the server's rate limiting regardless of the hash scheme.

**Dictionary attack after database theft**:

If an attacker steals the hash database (containing $EP_i$ and $S_i$ for all users), they must also know Pepper to compute any hash. Without Pepper:
- The attacker cannot verify any password guess: $H(P_{guess} \| S_i \| Pepper)$ cannot be computed without Pepper
- The entire hash database is cryptographically protected by the unknown Pepper — even with the hash list and salts, no offline attack is possible

If Pepper is also compromised (separate breach):
- The attacker now has all three: $EP_i$, $S_i$, $Pepper$
- Dictionary attack is possible per user, but the per-user salt means the attack must be conducted individually for each user — no batch efficiency
- The salt prevents two users with the same password from being cracked simultaneously

**Advantage over salt only**: if an attacker steals only the hash database but not the Pepper (stored separately in application code/config), the hashes are useless. Salt only provides no such protection — the salt is stored with the hash and is available to any attacker who reads the database.

**Rainbow tables**:

Even if Pepper is known, rainbow tables must be computed for each specific $(S_i, Pepper)$ combination. Since $S_i$ is unique per user and random, a separate rainbow table would need to be precomputed for every user — infeasible. The salt completely defeats rainbow tables even when Pepper is known.

If Pepper is unknown (only database stolen), rainbow tables are completely infeasible (the hash function is effectively keyed by the unknown Pepper — the table space is over $\{0,1\}^{|\text{Pepper}|}$ possible Peppers).

---

### Advantages of Salt + Pepper Combination

| Attack | Salt only | Pepper only | Salt + Pepper |
|---|---|---|---|
| Database stolen (hashes + salts) | Dictionary attack per user possible | Dictionary attack possible (Pepper known from code) | **Infeasible if Pepper not also stolen** |
| Database + Pepper stolen | — | All users vulnerable simultaneously (no per-user protection) | Must crack each user individually (per-user salt) |
| Rainbow tables, no Pepper | **Defeated by per-user salt** | Infeasible (unknown Pepper) | **Defeated by both** |
| Rainbow tables, Pepper known | — | Precomputable for specific Pepper | **Defeated by per-user salt** |
| Cross-user attacks | Not possible (per-user salt) | Possible (same hash for same password) | Not possible |
| Two data breaches needed | No | No | **Yes — requires both DB and Pepper** |

**Advantages**:
1. **Requires two separate breaches**: an attacker needs both the hash database (from DB server) AND the Pepper (from application server/code). If storage is properly segregated (database server ≠ application server), a single breach of either reveals nothing useful.
2. **Combines protections**: per-user salt defeats rainbow tables and cross-user attacks; Pepper defeats offline dictionary attacks on a stolen-but-Pepperless database.
3. **Defence in depth** (ch3.7 p.46): two independent layers of protection — one symmetric secret (Pepper) and one per-user randomness (salt).

**Drawbacks**:

1. **Pepper is a single point of failure**: if Pepper is compromised (code repository, config file leak, insider threat), all users' hashes become vulnerable to dictionary attacks (though the per-user salt still prevents cross-user and rainbow table attacks). Pepper rotation requires re-hashing all stored passwords — operationally disruptive.

2. **Not stored → cannot be recovered**: if the Pepper is lost (e.g., no backup of the application secret), all stored hashes become permanently unusable — all users must reset their passwords. This is an availability risk.

3. **Same Pepper for all users means correlated exposure**: unlike the per-user salt, the Pepper compromise affects all users simultaneously rather than being isolated to one account.

4. **Not a key in the formal sense**: $H(P_i \| S_i \| Pepper)$ uses $H$ as a one-way function, not a keyed MAC. An attacker who knows the construction can try offline attacks if they obtain Pepper. Using $HMAC_K(P_i \| S_i)$ with Pepper as the HMAC key would be more formally sound (ch2.2.3 p.63–66), but the practical difference for password storage is small if $H$ is a slow, salted function.

### Sources

- IS_UG_3_2_Appl_AuthMeth (p.11: salted password hashing, rainbow table prevention)
- IS_UG_2_2_3_SecM_HashMac (p.63–66: HMAC as formally keyed MAC construction)
- IS_UG_3_7_Appl_System (p.46: defence in depth; p.85: brute-force detection)

_Status: Complete_  
_Done by: William_
