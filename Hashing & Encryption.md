# HASHING
---
A Hashing Algorithm is a function that takes input(like a password) and converts it into fixed length string of characters, called hash value or digest.

It is a one way transformation:
```
Input: `"myPassword123"`
Output: `"5f4dcc3b5aa765d61d8327deb882cf99"` (example)
```

#### Key properties of hashing algorithms

A good hashing algorithm has these characteristics:
- **Deterministic**: Same input -> Same output every time.
- **Fixed Output Length:** No matter how big the input is.
- **Fast Computation**: It should be efficient to compute.
- **Pre-Image Resistance**: - Hard to reverse (We should not easily get the output from input hash).
- **Collision Resistance:** Hard to find two inputs with same hash.
- **Avalanche effect** → small change → completely different hash
#### Why hashing is important in a password manager

We never store raw passwords. Instead we store hashes:
- User enters password -> hash it -> compare it with stored hash.
- Even if our database leaks, attacker won't see actual passwords.

### Types of Hashing Algorithm
---
##### 1. Cryptographic Hash Functions
Designed for **security, integrity, and verification**

**Properties:**
- Deterministic
- Fixed output size
- Pre-image resistance
- Collision resistance
- Avalanche effect

**Examples:**
- **MD5** → Fast but broken (collision attacks)
- **SHA-1** → Deprecated (not secure)
- **SHA-256** → Secure, widely used (blockchain, SSL)

##### 2. Password Hashing / Key Derivation Functions (KDFs)
Designed to be **slow and resistant to brute-force attacks**

**Why slow?**  
→ Makes large-scale attacks impractical

**Examples:**
- **bcrypt**
    - Built-in salt
    - Adjustable cost factor
- **scrypt**
    - Memory + CPU intensive
    - Resists GPU attacks
- **Argon2**
    - Winner of Password Hashing Competition
    - Configurable (memory, time, parallelism)
    - Recommended today

##### 3. Non-Cryptographic Hash Functions
Designed for **speed, not security**

**Used in:**
- Hash tables
- Caching
- Load balancing

**Examples:**
- MurmurHash
- CityHash

>👉 _“For password storage, we should never use fast hash functions like SHA-256 directly. Instead, we use KDFs like bcrypt or Argon2 with salting and work factors.”_

### Salting
---
**Definition:**  
A **random value added to a password before hashing**

**How it works:**
```
hash = H(password + salt)
```
##### Why it’s used

**Problem without salt:** If two users have the same password:

```
password123 → same hash
```

Attackers can:
- Spot identical passwords
- Use **rainbow tables** (precomputed hash databases)

**With salt:**
```
password123 + randomSalt1 → hash1  
password123 + randomSalt2 → hash2
```

Now:
- Same password ≠ same hash
- Rainbow tables become useless
- Attackers must crack each password individually

**Key points:**
- Salt must be **unique per user**
- Salt is **stored in the database** (not secret)
- Modern algorithms like **bcrypt** and **Argon2** handle salting automatically

### Key Stretching
---
**Definition:**  
Making hashing intentionally **slow** by applying it multiple times or increasing computational cost

Example idea:
```
hash(hash(hash(hash(password))))
```

OR via cost parameters (better approach)

**Why it’s needed:**
- Slows down **brute-force attacks**
- Makes each guess expensive

**Key points:**
- Implemented via **work factor / cost**
- Built into:
    - **bcrypt** (cost factor)
    - **scrypt** (CPU + memory)
    - **Argon2** (time, memory, parallelism)

👉 Real-world idea:
- Fast hash (like SHA-256): millions/sec ❌
- bcrypt: maybe 100/sec ✅ (depends on cost)

### Peppering
---
**Definition:**  
A **secret value added to passwords before hashing**, similar to salt but **kept hidden**

**How it works:**
```
hash = H(password + salt + pepper)
```

**Why it’s needed:**
- Adds **extra layer of security**
- Even if DB is leaked → attacker still needs pepper

**Key points:**
- Pepper is **NOT stored in DB**
- Stored in:
    - Environment variables
    - Secrets manager
- Same pepper can be used for all users (or rotated)

---
### Key Differences
| Feature          | Salt 🧂                | Pepper 🌶️               |
| ---------------- | ---------------------- | ------------------------ |
| Secret?          | ❌ No                   | ✅ Yes                    |
| Stored in DB?    | ✅ Yes                  | ❌ No                     |
| Unique per user? | ✅ Yes                  | ❌ Usually same           |
| Purpose          | Prevent rainbow tables | Add extra security layer |

>“Passwords should be hashed using KDFs like bcrypt or Argon2, which include salting and key stretching. Salting prevents rainbow table attacks, key stretching slows brute-force attempts, and optionally peppering adds an extra secret layer stored outside the database.”


###  🌈 Rainbow Table Attack
---
A **precomputed lookup attack** used to reverse hashes.

Instead of hashing passwords on the fly, attackers:
1. Precompute hashes for millions/billions of common passwords
2. Store them in a table
3. When they get a leaked hash → just look it up

**How it works:**
```
password → hash
```

Attacker builds:
```
"123456" → hash1  
"password" → hash2  
"admin" → hash3  
```

Then if your DB leaks:
```
hash2 → instantly maps to "password"
```

No brute force needed. Just lookup.

##### Why it’s dangerous
- Extremely fast attack
- Works well on **unsalted hashes**
- Common passwords get cracked instantly

> _“Rainbow table attacks rely on precomputed hashes and are completely mitigated by proper salting.”_

### Side-Channel Attacks
---
Instead of breaking the algorithm mathematically, attackers exploit **information leakage from implementation**.
They observe _how_ a system behaves, not just _what_ it outputs.

##### Common types

###### ⏱️ Timing Attack
- Measure how long operations take
- Example: password comparison exits early → leaks info

```
if (input[i] != stored[i]) return false;
```

Faster failure = attacker learns correct prefix

###### 🔌 Power Analysis
- Used in hardware (smart cards, embedded systems)
- Power consumption reveals operations

###### 📡 Cache / Memory Attacks
- Observe memory access patterns
- Can leak sensitive data

###### 💥 Why it’s dangerous
- Works even if algorithm is **cryptographically secure**
- Targets **implementation flaws**, not theory

> _“Side-channel attacks exploit implementation leaks like timing or memory access, so defenses focus on constant-time operations and secure implementations.”_

## SHA-256
---
SHA-256 takes **any input → produces a fixed 256-bit hash**.

👉 Design goals:
- One-way (pre-image resistant)
- Collision resistant
- Deterministic
- Avalanche effect

#### High-Level Flow
Think of SHA-256 like a pipeline:

```
Input → Padding → Blocks → Compression Rounds → Final Hash
```

Think of it like a **meat grinder** — you can put anything in, but you can never reconstruct the original from what comes out.

```
"hello"        →  2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
"hello!"       →  ce06092fb948d9af2d6f72c0a30d37b6b3f0d176c5b4bcf5a5143f2b89bfb40
"hello world"  →  b94d27b9934d3e08a52e52d7da7dabfac484efe04294e576ca0588e1d2b21c4c
```

Notice — changing even **one character** completely changes the output.

#### Key Properties
|Property|Meaning|
|---|---|
|**Fixed output**|Always 256 bits / 64 characters, no matter the input size|
|**One-way**|You cannot reverse it back to the original|
|**Deterministic**|Same input always gives same output|
|**Avalanche effect**|Tiny change in input = completely different output|
|**Collision resistant**|Nearly impossible for two different inputs to produce the same hash|

##### Why SHA-256 is Fast
- Uses simple CPU operations (bitwise ops)
- No memory hardness
- Optimized for performance

👉 That’s why:  
❌ Bad for passwords  
✅ Great for integrity

#### SHA Use Cases:
- Data integrity (file checks)
- Digital signatures
- HMAC (API signing)
- Pre-hashing (advanced cases)
- Key derivation building blocks

### Limitations
- Fast → vulnerable to brute-force (for passwords)
- Needs salting externally
- No built-in cost factor

👉 That’s why we use:
- bcrypt
- Argon2
for password storage


>“SHA-256 processes input in 512-bit blocks, applies padding, and runs each block through 64 rounds of bitwise transformations using functions like Ch and Maj. It maintains an internal state of 8 words and produces a 256-bit hash. It’s designed to be fast and collision-resistant, making it suitable for integrity but not for password hashing.”

#### Go Implementation
---
```go
package main

import (
	"crypto/sha256"  
	"encoding/hex"
	"fmt"
)

func hashSHA256(input string) string {
	hash := sha256.Sum256([]byte(input))
	return hex.EncodeToString(hash[:])
}

func main(){
	fmt.Println(hashSHA256("password123"))
}

```

#### TypeScript (Node.js) Implementation
---
```typescript
import crypto from "crypto";

export function sha256(input: string): string {
	return crypto
			.createHash("sha256")
			.update(input)
			.digest("hex")
}
```


## bcrypt
---
**A password hashing algorithm with built-in salting + key stretching**

#### What problem does bcrypt solve?

👉 **Fast hashes (like SHA-256) are bad for passwords**
- Attackers can try **millions of guesses per second**
- Even with salt → still brute-forceable

👉 bcrypt solves this by being:
- **Slow (intentionally)**
- **Adaptive (cost can increase over time)**

##### Core Idea
bcrypt =  **hashing + salting + key stretching (built-in)**

####  Does bcrypt use salt?

👉 Yes—and this is important.
- Automatically generates a **random salt per password**
- Stores it as part of the final hash

Example (conceptual):
```
$2b$10$<salt><hash>
```

- `10` → cost factor
- Salt is embedded
- Hash is stored together

> “bcrypt automatically handles salting, so developers don’t need to manage it separately.”

#### What is the cost factor?
This is the **most important concept in bcrypt**.

👉 Cost controls how slow hashing is.
cost = 10 → 2^10 iterations  
cost = 12 → 2^12 iterations

👉 Increasing cost:
- Makes hashing slower
- Makes brute-force harder

#### Real-world intuition
|Cost|Approx Time|
|---|---|
|8|Very fast (weak)|
|10|Acceptable|
|12|Good standard|
|14+|High security|
A bcrypt hash looks like:
```
$2b$12$KIXQ4hFz8FJd9l5Yp5K7Fe3JzR9r0kYq1xYpQJ7Fz3XQ6VQy5e7eG
```

Breakdown:

| Part                     | Meaning           |
| ------------------------ | ----------------- |
| `$2b$`                   | Algorithm version |
| `12`                     | Cost factor       |
| `KIXQ4hFz8FJd9l5Yp5K7Fe` | Salt              |
| rest                     | Actual hash       |

👉 Everything is stored in **one string**
#### Why is bcrypt secure?
 **✅ Slow hashing**
- Limits brute-force speed

 **✅ Built-in salt**
- Prevents rainbow table attacks

 **✅ Adaptive cost**
- Future-proof against faster hardware

#### Limitations of bcrypt (important!)
- **Not memory-hard**
	- Can still be optimized on GPUs
That’s why **Argon2** is considered better today

-  **Input length limit (~72 bytes)**
	- Longer passwords get truncated

- **Slower but not enough for future threats**
	- Modern recommendation is shifting toward Argon2


#### bcrypt vs SHA-256
| Feature        | SHA-256     | bcrypt   |
| -------------- | ----------- | -------- |
| Speed          | Very fast ⚡ | Slow 🐢  |
| Salt           | Manual      | Built-in |
| Cost factor    | ❌           | ✅        |
| Password safe? | ❌           | ✅        |

> “SHA-256 is designed for speed and integrity, while bcrypt is designed for slow password hashing with built-in salting and cost control.”

#### NOTE:
> bcrypt is **intentionally non-deterministic** for its output — it generates a random salt each time:

```
bcrypt("hello") → $2a$10$N9qo8uLOickgx2ZMRZo... (random salt baked in)
bcrypt("hello") → $2a$10$XvjDy3KqHB72kGQVH8k... (different every time)
```

> That's why we use `CompareHashAndPassword` instead of hashing again and comparing — bcrypt extracts the stored salt to reproduce the check.

#### When should you use bcrypt?
👉 Use bcrypt when:
- You need **secure password storage**
- Simplicity is preferred
- Argon2 is not available

👉 Prefer **Argon2** when:
- You want modern, stronger protection

>“bcrypt is a password hashing function that incorporates salting and key stretching with an adjustable cost factor to make brute-force attacks computationally expensive.”

#### Go Implementation (Production Style)
##### Hash Password
---
```go
package main

import (
	"fmt"
	"goland.org/x/crypto/bcrypt"
)

func HashPassword(password string) (string, error){
	hashed, err := bcrypt.GenerateFromPassword(
		[]byte(password),
		bcrypt.DefaultCost // usually 10-12
	)
	
	return string(hashed),err
}
```

##### Verify Password
---
```go
func checkPassword(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword(
		[]byte(hash),
		[]byte(password),
	)
	
	return err == nil
}
```

### 💻 TypeScript (Node.js)
##### Hash Password
---

```ts
import bcrypt from "bcrypt";

export async function hashPassword(password: string) {
  const saltRounds = 12;
  return await bcrypt.hash(password, saltRounds);
}
```

##### Verify Password
---
```ts
export async function verifyPassword(  
  password: string,  
  hash: string  
) {  
  return await bcrypt.compare(password, hash);  
}
```

# Argon2
---
**Argon2** is a password hashing and key derivation algorithm designed to be secure by making attackers spend significant CPU time and memory for every password guess.

#### Why was Argon2 created?
bcrypt was good, but had limitations:
- Not memory-hard
- Vulnerable to GPU/ASIC optimizations

👉 Argon2 was designed to fix this.

#### Argon2 Variants

|Variant|Use Case|
|---|---|
|Argon2d|GPU-resistant (but side-channel unsafe)|
|Argon2i|Side-channel safe (but weaker vs GPU)|
|**Argon2id**|✅ Hybrid → **BEST choice**|
> A **side-channel attack** exploits how a system computes something - timing, power usage, memory access patterns - rather than breaking the algorithm mathematically.
> 
> **THE CORE IDEA** Instead of cracking the hash directly, an attacker observes physical or behavioral leakage from the system during computation.

#### Argon2d
---
In Argon2d, the way memory is accessed depends on the password itself. This makes the process unpredictable and hard to optimize on GPUs. However, because the access pattern depends on secret data, an attacker observing the system might gain information through timing or cache behavior.

##### Argon2i
---
Argon2i avoids leaking information by always accessing memory in a predictable way, regardless of the password. This makes it safe against side-channel attacks, but since the pattern is predictable, attackers can optimize their hardware and potentially reduce memory usage using trade-offs.

##### Argon2id
---
Argon2id is a hybrid variant of Argon2 that combines the side-channel resistance of Argon2i with the GPU resistance of Argon2d, making it the recommended choice for most applications.

Argon2id starts with a predictable memory access pattern to avoid leaking sensitive information, and then switches to a password-dependent pattern to prevent GPU optimizations. This gives us the best of both worlds—security against both side-channel and brute-force attacks.

#### Core Idea
Argon2 =  
👉 **Slow + Memory-Hard + Configurable**

Most important word here:
👉 **Memory-hard**

##### What does memory-hard mean?
It requires **a lot of RAM** to compute the hash.

👉 Why this matters:
- GPUs are great at parallel compute
- But **memory is expensive and limited per core**

👉 Result:
- Attackers can’t scale easily
- Brute-force becomes much harder

#### Key Parameters
- **Time Cost (t)**
	- Number of iterations
	- More iterations → slower
- **Memory Cost (m)**
	- Amount of RAM used
	- More memory → harder for attackers
- **Parallelism (p)**
	- Number of threads
	- Controls CPU parallel usage

> “Argon2 allows tuning of time, memory, and parallelism, making it adaptable to different security requirements.”

#### Does Argon2 use salt?
Yes (like bcrypt)
- Automatically uses a **unique salt**
- Prevents rainbow table attacks

#### What about pepper?
👉 Not built-in, but we can add it externally

#### Why is Argon2 better than bcrypt?
|Feature|bcrypt|Argon2|
|---|---|---|
|Memory-hard|❌|✅|
|GPU resistance|Medium|High|
|Configurable|Limited|Highly|
|Modern standard|❌|✅|
**Key takeaway:**
> bcrypt slows attackers  
> Argon2 **slows + limits their hardware advantage**

###### SUMMARY:
>“Argon2 is a modern password hashing algorithm that improves on bcrypt by introducing memory hardness and configurable parameters, making it resistant to GPU-based brute-force attacks.”

#### Implementation:

###### GO:
---
```go
package password

import (
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
	"errors"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

type Params struct {
	Memory      uint32
	Iterations  uint32
	Parallelism uint8
	SaltLength  uint32
	KeyLength   uint32
}

var DefaultParams = &Params{
	Memory:      64 * 1024, // 64 MB
	Iterations:  3,
	Parallelism: 4,
	SaltLength:  16,
	KeyLength:   32,
}

// GenerateHash creates a hash in PHC string format
func GenerateHash(password string, p *Params) (string, error) {
	salt, err := generateRandomBytes(p.SaltLength)
	if err != nil {
		return "", err
	}

	hash := argon2.IDKey(
		[]byte(password),
		salt,
		p.Iterations,
		p.Memory,
		p.Parallelism,
		p.KeyLength,
	)

	b64Salt := base64.RawStdEncoding.EncodeToString(salt)
	b64Hash := base64.RawStdEncoding.EncodeToString(hash)

	// PHC format
	encoded := fmt.Sprintf(
		"$argon2id$v=19$m=%d,t=%d,p=%d$%s$%s",
		p.Memory,
		p.Iterations,
		p.Parallelism,
		b64Salt,
		b64Hash,
	)

	return encoded, nil
}

func ComparePassword(password, encodedHash string) (bool, error) {
	p, salt, hash, err := decodeHash(encodedHash)
	if err != nil {
		return false, err
	}

	newHash := argon2.IDKey(
		[]byte(password),
		salt,
		p.Iterations,
		p.Memory,
		p.Parallelism,
		p.KeyLength,
	)

	if subtle.ConstantTimeCompare(hash, newHash) == 1 {
		return true, nil
	}
	return false, nil
}

// --- helpers ---

func generateRandomBytes(n uint32) ([]byte, error) {
	b := make([]byte, n)
	_, err := rand.Read(b)
	return b, err
}

func decodeHash(encoded string) (*Params, []byte, []byte, error) {
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 {
		return nil, nil, nil, errors.New("invalid hash format")
	}

	var version int
	_, err := fmt.Sscanf(parts[2], "v=%d", &version)
	if err != nil || version != 19 {
		return nil, nil, nil, errors.New("incompatible version")
	}

	p := &Params{}
	_, err = fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d",
		&p.Memory, &p.Iterations, &p.Parallelism)
	if err != nil {
		return nil, nil, nil, err
	}

	salt, err := base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return nil, nil, nil, err
	}
	p.SaltLength = uint32(len(salt))

	hash, err := base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return nil, nil, nil, err
	}
	p.KeyLength = uint32(len(hash))

	return p, salt, hash, nil
}
```

👉 **Argon2 doesn’t auto-generate the salt for you in low-level APIs like Go’s `argon2.IDKey`.**  
👉 It **uses** a salt, but **you must provide it**.

###### TypeScript:
---
```ts
import argon2 from "argon2";  
  
export async function hashPassword(password: string) {  
  return await argon2.hash(password, {  
    type: argon2.argon2id,  
    memoryCost: 65536,  
    timeCost: 3,  
    parallelism: 4,  
  });  
}
```

```ts
export async function verifyPassword(password: string, hash: string) {  
  return await argon2.verify(hash, password);  
}
```

# Encryption
---
Encryption = converting **plaintext → ciphertext** using a **key**
```
plaintext + key → ciphertext
```

Goal:
- Keep data **confidential** (only authorized parties can read it)

#### Encryption vs Hashing
| Feature    | Hashing    | Encryption      |
| ---------- | ---------- | --------------- |
| Reversible | ❌ No       | ✅ Yes           |
| Key used   | ❌ No       | ✅ Yes           |
| Purpose    | Integrity  | Confidentiality |
| Output     | Fixed Size | Variable        |
> “Hashing is one-way, while encryption is reversible using a key.”

### Types of Encryption
#### Symmetric Encryption
Same key for encryption + decryption
```
Encrypt(key, data) → ciphertext  
Decrypt(key, ciphertext) → data
```

##### Example:
👉 **Advanced Encryption Standard(AES)**
- Fast ⚡
- Used everywhere (TLS, databases, disk encryption)

#### Asymmetric Encryption
Two keys:
- Public key (encrypt)
- Private key (decrypt)

**Example:**
👉 **RSA**

#### ⚡ Key Difference
| Feature | Symmetric | Asymmetric   |
| ------- | --------- | ------------ |
| Speed   | Fast ⚡    | Slow 🐢      |
| Keys    | 1         | 2            |
| Use     | Bulk data | Key exchange |

> “Asymmetric encryption is mainly used for key exchange, while symmetric encryption is used for actual data encryption.”


#### How Real Systems Work
Real-world systems use **both**:
###### Example: HTTPS
1. Use **RSA** (or similar) to securely share a key
2. Use **AES** for fast communication

👉 Hybrid encryption model

### Block Cipher vs Stream Cipher
---
#### Block Cipher
👉 Encrypts **fixed-size blocks** (e.g., 128 bits)

 **Example:**
👉 Advanced Encryption Standard

###### Properties:
- Deterministic (same input → same output with same key)
- Needs **modes of operation** (CBC, GCM, etc.)

#### 🌊 Stream Cipher
👉 Encrypts **data bit-by-bit or byte-by-byte**

**Example:**
👉 ChaCha20

###### Properties:
- Generates a **keystream**
- Combined with plaintext using XOR

#### ⚡ Key Difference
|Feature|Block Cipher|Stream Cipher|
|---|---|---|
|Unit|Blocks|Continuous stream|
|Needs mode?|Yes|No|
|Speed|Moderate|Very fast|

> “Block ciphers operate on fixed-size blocks and require modes, while stream ciphers generate a keystream and encrypt data continuously.”

#### Modes of Operation
Block ciphers alone are not enough.
#### ❌ ECB
- Same plaintext → same ciphertext  
    👉 leaks patterns → insecure

---
#### ⚠️ CBC
- Uses IV
- Chains blocks together
- No authentication ❌

---
#### ✅ GCM (Modern Standard)

👉 AEAD mode
- Encryption + integrity
- Fast + secure

> “Never use ECB; prefer AEAD modes like GCM.”

### IV vs Nonce
---
##### 🔹 IV (Initialization Vector)
- Random value
- Used in modes like CBC

---

##### 🔹 Nonce (Number used once)
- Must be **unique (not necessarily random)**
- Used in modern modes like GCM, ChaCha20

#### ⚠️ Key Difference

| Feature        | IV     | Nonce               |
| -------------- | ------ | ------------------- |
| Requirement    | Random | Unique              |
| Reuse allowed? | ❌ No   | ❌ No                |
| Used in        | CBC    | GCM, stream ciphers |
> “A nonce must be unique per encryption, while an IV is typically random—reusing either can break security.”

#### Padding (important for block ciphers)

Block ciphers need fixed-size input.
👉 If data isn’t aligned → padding is added

**Example:**
- PKCS#7 padding

⚠️ Why important?
- Incorrect padding handling → vulnerabilities (padding oracle attacks)

#### Keystream (for stream ciphers)

Stream ciphers work like:
```
ciphertext = plaintext ⊕ keystream
```

👉 If keystream is reused:
- Security breaks instantly ❌

#### Key Management (VERY IMPORTANT)

Encryption is only as strong as:  
👉 **how you manage keys**
##### Includes:
- Key generation (secure randomness)
- Key storage (KMS, env, HSM)
- Key rotation
- Access control

> “Most real-world crypto failures come from poor key management, not weak algorithms.”

#### Common Pitfalls (high-value)
- ❌ Reusing nonce
- ❌ Using ECB
- ❌ No authentication (just encryption)
- ❌ Hardcoding keys
- ❌ Rolling your own crypto

## AEAD — The Core Abstraction
---
**Authenticated Encryption with Associated Data**

AEAD gives you **everything you want in one primitive**:
- 🔒 Confidentiality (encryption)
- ✅ Integrity (no tampering)
- 🔑 Authentication (trusted source)

##### Mental Model
```
ciphertext, tag = AEAD_Encrypt(key, nonce, plaintext, associated_data)
plaintext = AEAD_Decrypt(key, nonce, ciphertext, associated_data, tag)
```

#### What is “Associated Data”?
This is one of the most important concepts.

👉 Data that is:
- **NOT encrypted**
- BUT **must be authenticated**

 **Example**
- HTTP headers
- User ID
- Metadata

If attacker modifies it → decryption fails ❌

> “AEAD allows binding metadata to ciphertext without encrypting it.”

#### Key Components of AEAD
---
##### 1. Key
- Secret
- Shared between parties
##### 2. Nonce (VERY IMPORTANT)
- Unique per encryption
- Prevents reuse attacks

👉 Reusing nonce = catastrophic ❌
##### 3. Plaintext
- Data you want to encrypt
#### 4. Ciphertext
- Encrypted output
##### 5. Authentication Tag
- Ensures integrity
- Detects tampering

#### AEAD Workflow (Step-by-Step)

##### Encryption:
1. Take plaintext
2. Mix with key + nonce
3. Produce:
    - ciphertext
    - authentication tag
##### Decryption:
1. Verify tag
2. If valid → decrypt
3. If not → reject

👉 Important:

> “Decryption should never happen before authentication.”


#### Popular AEAD Schemes
---
#### 🔹 AES-GCM
👉 Based on **Advanced Encryption Standard**

- Very fast (hardware accelerated)
- Widely used (TLS, HTTPS)

#### 🔹 ChaCha20-Poly1305
👉 Uses **Poly1305**
- Better on mobile / low-end CPUs
- Resistant to timing attacks

#### 🔹 XChaCha20-Poly1305
- Extended nonce version
- Safer nonce handling
- Great for distributed systems


##### AES-GCM vs ChaCha20-Poly1305
|Feature|AES-GCM|ChaCha20-Poly1305|
|---|---|---|
|Speed|Fast (with hardware)|Fast (software)|
|Platform|CPUs with AES-NI|Mobile / IoT|
|Nonce size|96-bit|96-bit (XChaCha: 192-bit)|

>“In modern systems, we never use raw encryption primitives. We use AEAD constructions like AES-GCM or ChaCha20-Poly1305 to ensure both confidentiality and integrity in a single operation.”

