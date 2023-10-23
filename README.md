# `oslo`

> This package is _highly_ experimental - use at your own risk

A collection of utilities for auth, including:

- [`oslo/cookie`](#oslocookie): Cookie parsing and serialization
- [`oslo/encoding`](#osloencoding): Encode base64, base64url, base32, hex
- [`oslo/oauth2`](#oslooauth2): OAuth2 helpers
  - [`oslo/oauth2/providers`](#oslooauth2providers): Built in OAuth2 providers (Apple, Github, Google)
- [`oslo/otp`](#oslootp): HOTP, TOTP
- [`oslo/password`](#oslopassword): Password hashing
- [`oslo/random`](#oslorandom): Generate cryptographically random strings and numbers
- [`oslo/token`](#oslotoken): Verification tokens (email verification, password reset, etc)
- [`oslo/request`](#oslorequest): CSRF protection
- [`oslo/session`](#oslosession): Session management
- [`oslo/webauthn`](#oslowebauthn): Verify Web Authentication API attestations and assertions

Aside from `oslo/password`, every module works in any environment, including Node.js, Cloudflare Workers, Deno, and Bun.

## Installation

```
npm i oslo
pnpm add oslo
yarn add oslo
```

## `oslo/cookie`

```ts
import { serializeCookie } from "oslo/cookie";

const cookie = serializeCookie(name, value, {
	// optional
	expires: new Date(),
	maxAge: 60 * 60,
	path: "/",
	httpOnly: true,
	secure: true,
	sameSite: "lax"
});
response.headers.set("Set-Cookie", cookie);

// names and values are URI component encoded
serializeCookie("this is fine =;", "this too =;");
```

```ts
import { parseCookieHeader } from "oslo/cookie";

// returns Map<string, string>
const cookies = parseCookieHeader("cookie1=hello; cookie2=bye");
```

## `oslo/encoding`

```ts
import { encodeBase64, decodeBase64 } from "oslo/encoding";

const textEncoder = new TextEncoder();
const encoded = encodeBase64(textEncoder.encode("hello world"));
const decoded = decodeBase64(encoded);

import { encodeBase64url, decodeBase64url } from "oslo/encoding";
import { encodeHex, decodeHex } from "oslo/encoding";
import { encodeBase32, decodeBase32 } from "oslo/encoding";
```

## `oslo/oauth2`

```ts
import { OAuth2Controller } from "oslo/oauth2";

const authorizeEndpoint = "https://github.com/login/oauth/authorize";
const tokenEndpoint = "https://github.com/login/oauth/access_token";

const oauth2Controller = new OAuth2Controller(clientId, authorizeEndpoint, tokenEndpoint, {
	// optional
	redirectURI: "http://localhost:3000/login/callback"
});
```

```ts
import { generateState, generateCodeVerifier } from "oslo/oauth2";

const state = generateState();
const codeVerifier = generateCodeVerifier(); // for PKCE flow

const url = await createAuthorizationURL({
	// optional
	state,
	scope: ["user:email"],
	codeVerifier
});
```

```ts
import { verifyState, AccessTokenRequestError } from "oslo/oauth2";

if (!verifyState(storedState, state)) {
	// error
}

// ...

try {
	const { accessToken, refreshToken } = await oauth2Controller.validateAuthorizationCode<{
		refreshToken: string;
	}>(code, {
		// optional
		credentials: clientSecret,
		authenticateWith: "request_body", // default: "http_basic_auth"
		codeVerifier // for PKCE flow
	});
} catch (e) {
	if (e instanceof AccessTokenRequestError) {
		// see https://www.rfc-editor.org/rfc/rfc6749#section-5.2
		const { request, message, description } = e;
	}
	// unknown error
}
```

### `oslo/oauth2/providers`

Some providers may require a code challenge and code verifier.

```ts
import { Github, Apple, Google } from "oslo/oauth2/providers";

const githubOAuth = new Github(clientId, clientSecret, {
	scope: ["user:email"]
});

const url = await githubOAuth.createAuthorizationURL(state);

const tokens = await githubOAuth.validateAuthorizationCode(code);
```

## `oslo/oidc`

```ts
import { parseIdToken } from "oslo/oidc";

const { sub, exp, email } = parseIdToken<{ email: string }>(idToken);
```

## `oslo/otp`

```ts
import { generateHOTP } from "oslo/otp";

const secret = new Uint8Array(20);
crypto.getRandomValues(secret);

let counter = 0;

const otp = await generateHOTP(secret, counter); // default 6 digits
const otp = await generateHOTP(secret, counter, 8); // 8 digits (max)
```

```ts
import { TOTPController } from "oslo/otp";
import { TimeSpan } from "oslo";

const totpController = new TOTPController({
	// optional
	period: new TimeSpan(30, "s"), // default: 30s
	digits: 6 // default: 6
});

const secret = new Uint8Array(20);
crypto.getRandomValues(secret);

const otp = await totpController.generate(secret);
const validOTP = await totpController.verify(otp, secret);
```

```ts
import { createHOTPKeyURI, createTOTPKeyURI } from "oslo/otp";

const secret = new Uint8Array(20);
crypto.getRandomValues(secret);

const issuer = "My website";
const accountName = "user@example.com";

const uri = createHOTPKeyURI(issuer, accountName, secret, {
	//optional
	counter: 0, // default: 0
	algorithm: "SHA-1", // default: SHA-1
	digits: 6 // default: 6
});

const uri = createTOTPKeyURI(issuer, accountName, secret, {
	//optional
	period: new TimeSpan(30, "s"), // default: 30s
	algorithm: "SHA-1", // default: SHA-1
	digits: 6 // default: 6
});
```

## `oslo/password`

Hash passwords with argon2id, scrypt, and bcrypt using the fastest package available for Node.js.

```ts
import { Argon2id, Scrypt, Bcrypt } from "oslo/password";

// `Scrypt` and `Bcrypt` implement the same methods
const argon2id = new Argon2id(options);
const hash = await argon2id.hash(password);
const matches = await argon2id.verify(hash, password);
```

This specific module only works in Node.js. See these packages for other runtimes:

- [`hash-wasm`](https://github.com/Daninet/hash-wasm): Pure WASM implementation
- [`argon2`](https://deno.land/x/argon2@v0.9.2): Rust-based Deno package

## `oslo/random`

All functions are cryptographically secure.

```ts
import { generateRandomString, alphabet } from "oslo/random";

const id = generateRandomString(16, alphabet("0-9", "a-z", "A-Z")); // alphanumeric
const id = generateRandomString(16, alphabet("0-9", "a-z")); // alphanumeric (lowercase)
const id = generateRandomString(16, alphabet("0-9")); // numbers only
const id = generateRandomString(16, alphabet("0-9", "a-z", "A-Z", "-", "-")); // alphanumeric with `_` and`-`
```

```ts
import { random } from "oslo/random";

const num = random(); // cryptographically secure alternative to `Math.random()`
```

```ts
import { generateRandomNumber } from "oslo/random";

// random integer between 0 (inclusive) and 10 (exclusive)
const num = generateRandomNumber(0, 10);
```

## `oslo/request`

CSRF protection.

```ts
import { verifyRequestOrigin } from "oslo/request";

// only allow same-origin requests
const validRequestOrigin = verifyRequestOrigin(request.headers.get("Origin"), {
	host: request.headers.get("Host")
});
const validRequestOrigin = verifyRequestOrigin(request.headers.get("Origin"), {
	host: request.url
});

if (!validRequestOrigin) {
	// invalid request origin
	return new Response(null, {
		status: 400
	});
}
```

```ts
// true
verifyRequestOrigin("https://example.com", "example.com");

// true
verifyRequestOrigin("https://foo.example.com", "bar.example.com", {
	allowedSubdomains: "*" // wild card to allow any subdomains
});

// true
verifyRequestOrigin("https://foo.example.com", "bar.example.com", {
	allowedSubdomains: ["foo"]
});

// true
verifyRequestOrigin("https://example.com", "foo.example.com", {
	allowBaseDomain: true
});
```

## `oslo/session`

```ts
import { SessionController } from "oslo/session";
import { generateRandomString, alphabet } from "oslo/random";
import { TimeSpan } from "oslo";

import type { Session } from "oslo/session";

// implements sliding window expiration
// expires in 30 days unless used within 15 days before expiration (1/2 of expiration)
// in which case the expiration gets pushed back another 30 days
const sessionController = new SessionController(new TimeSpan(30, "d"));

async function validateSession(sessionId: string): Promise<Session | null> {
	const databaseSession = await db.getSession(sessionId);
	if (!databaseSession) {
		return null;
	}
	const session = sessionController.validateSessionState(sessionId, databaseSession.expires);
	if (!session) {
		await db.deleteSession(sessionId);
		return null;
	}
	if (session.fresh) {
		// session expiration was extended
		await db.updateSession(session.sessionId, {
			expires: session.expiresAt
		});
	}
	return session;
}

async function createSession(): Promise<Session> {
	const sessionId = generateRandomString(41, alphabet("a-z", "A-Z", "0-9"));
	const session = sessionController.createSession(sessionId);
	await db.insertSession({
		// you can store any data you want :D
		id: session.sessionId,
		expires: session.expiresAt
	});
	return session;
}

const sessionCookieController = sessionController.sessionCookieController(cookieName, {
	// optional
	// if the cookie expires
	expires: true, // default: true
	// set to `false` in dev
	secure: true, // default: false
	path: "/", // default: "/"
	domain, // default: undefined
	sameSite // default: "lax"
});
```

```ts
const session = await createSession();
// store session
const cookie = sessionCookieController.createSessionCookie(session.sessionId);
```

```ts
// get cookie
const sessionId = sessionCookieController.parseCookieHeader(headers.get("Cookie"));
const sessionId = cookies.get(sessionCookieController.cookieName);

if (!sessionId) {
	// 401
}
const session = await validateSession(sessionId);
if (session.fresh) {
	// session expiration was extended
	const cookie = sessionCookieController.createSessionCookie(session.sessionId);
	// set cookie again
}
```

```ts
// delete session by overriding the current one
const cookie = sessionCookieController.createBlankSessionCookie();
```

## `oslo/token`

For email verification tokens and password reset tokens.

```ts
import { VerificationTokenController } from "oslo/token";
import { isWithinExpirationDate } from "oslo";
import { generateRandomString, alphabet } from "oslo/random";

import type { Token } from "oslo/token";

// expires in 2 hours
const verificationTokenController = new VerificationTokenController(new TimeSpan(2, "h"));

async function generatePasswordResetToken(userId: string): Promise<Token> {
	// check if an unused token already exists
	const storedUserTokens = await db
		.table("password_reset_token")
		.where("user_id", "=", userId)
		.getAll();
	if (storedUserTokens.length > 0) {
		// if exists, check the expiration
		const reusableStoredToken = storedUserTokens.find((token) => {
			// returns true if there's 1 hour left (1/2 of 2 hours)
			return verificationTokenController.isTokenReusable(token.expires);
		});
		// reuse token if it exists
		if (reusableStoredToken) return reusableStoredToken.id;
	}
	// generate a new token and store it
	const token = verificationTokenController.createToken(
		generateRandomString(63, alphabet("a-z", "0-9")),
		userId
	);
	await db
		.insertInto("password_reset_token")
		.values({
			id: token.value,
			expires: token.expiresAt,
			user_id: token.userId
		})
		.executeTakeFirst();
	return token;
}

async function validatePasswordResetToken(token: string): Promise<string> {
	const storedToken = await db.transaction().execute(async (trx) => {
		// get token from db
		const storedToken = await trx.table("password_reset_token").where("id", "=", token).get();
		if (!storedToken) return null;
		await trx.table("password_reset_token").where("id", "=", token).delete();
		return storedToken;
	});
	if (!storedToken) throw new Error("Invalid token");
	// check for expiration
	if (!isWithinExpirationDate(storedToken.expires)) {
		throw new Error("Expired token");
	}
	// return owner
	return storedToken.user_id;
}
```

```ts
const token = await generatePasswordResetToken(session.userId);
// send password reset email with link (page with form for inputting new password)
await sendEmail(`http://localhost:3000/reset-password/${token.value}`);
```

## `oslo/webauthn`

`validateAttestationResponse()` does not validate attestation certificates. `validateAssertionResponse()` currently only supports ECDSA using secp256k1 curve and SHA-256 (algorithm ID `-7`).

```ts
import { WebAuthnController } from "oslo/webauthn";

const webauthn = new WebAuthnController("http://localhost:3000");

try {
	const response: AttestationResponse = {
		// all `ArrayBufferLike` type (`Uint8Array`, `ArrayBuffer` etc)
		clientDataJSON,
		authenticatorData
	};
	await webauthn.validateAttestationResponse(response, challenge);
} catch {
	// failed to validate
}

try {
	const response: AssertionResponse = {
		// all `ArrayBufferLike` type (`Uint8Array`, `ArrayBuffer` etc)
		clientDataJSON,
		authenticatorData,
		signature
	};
	await webauthn.validateAssertionResponse(
		"ES256",
		response,
		publicKey, // `ArrayBufferLike`
		challenge // `ArrayBufferLike`
	);
} catch {
	// failed to validate
}
```
