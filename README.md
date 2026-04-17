# FlowOrca — Drawbridge
### Drawbridge

```
                        |>>>                        |>>>
                        |                           |
                    _  _|_  _                   _  _|_  _
                   |;|_|;|_|;|                 |;|_|;|_|;|
                   \\..      /                 \\.    .  /
                    \\.  ,  /                   \\:  .  /
                     ||:   |_   _   _   _   _   _||:   |
                     ||:  .|||_|;|_|;|_|;|_|;|_|;||:.  |
                     ||:   ||.    .     .      . ||:  .|
                     ||: . || .     . .   .  ,   ||:   |
                     ||:   ||     FlowOrca     . ||: , |
                     ||:   ||:  ,  _______   .   ||:   |
                     ||:   || .   /+++++++\    . ||:   |
                     ||:   ||.    |+++++++| .    ||: . |
                     ||: . ||: ,  |+++++++|.  . _||_   |
                     '-- ~~_ _ .  |+++++++|-- ~   ~~- --,
                                      
```

```
# curl -s https://www.floworca.com | grep -c "plaintext"
0
```

Encrypted gateway for pre-release and private projects. Static site on GitHub Pages. No backend, no plaintext anywhere in the repo or on the wire.

## What this is

The repo is public but the content behind the gate isn't. The whole inner app is AES-256-GCM encrypted, base64 encoded, and stuffed into a single blob inside index.html. GitHub serves ciphertext. view-source shows ciphertext. Crawlers get ciphertext. There's no server to compromise because there is no server.

## How the crypto works

```
passphrase
  |
  +---> PBKDF2 (SHA-512, 600k rounds, 16-byte salt)
  |       |
  |       +---> 256-bit AES key
  |               |
  |               +---> AES-GCM decrypt (12-byte IV, 16-byte auth tag)
  |                       |
  |                       +---> raw HTML
  |                               |
  |                               +---> document.write() -- gate replaced
  |
  +---> wrong key? "Key denied." -- no oracle, no timing leak
```

All runs in the browser via Web Crypto API. No libraries, no dependencies, no external JS.

## The blob

The encrypted payload lives in a const called E. Base64 decode it and you get:

```
 0                16              28           44
 +----------------+---------------+------------+----------------------+
 |     salt       |      iv       |    tag     |      ciphertext      |
 |   (16 bytes)   |  (12 bytes)   | (16 bytes) |    (variable len)    |
 +----------------+---------------+------------+----------------------+
```

- **salt** -- 16 random bytes. Fed to PBKDF2 with the passphrase to derive the key. Same passphrase + different salt = different key.
- **iv** -- 12-byte nonce for AES-GCM. Makes sure the same plaintext encrypts differently each time.
- **tag** -- 16-byte auth tag. Tamper with anything and decryption fails outright.
- **ciphertext** -- the actual app. HTML, CSS, JS, all of it. Useless without the key.

## Decryption steps

```
1.  base64 decode E
2.  slice out salt, iv, tag, ciphertext
3.  import passphrase into PBKDF2
4.  derive AES-256-GCM key (SHA-512, 600k iterations)
5.  concat ciphertext + tag
6.  decrypt with AES-GCM using the iv
7.  decode to UTF-8
8.  document.write() -- gate is gone, app takes over
```

600k PBKDF2 rounds makes brute-force slow. GCM gives you encryption and integrity in one shot. Wrong key throws, no garbage output, just "Key denied."

## What happens to the page

When decryption works, the gate nukes itself:

```
document.open()   -- wipe the DOM
document.write()  -- write the decrypted app
document.close()  -- done
```

Everything from the gate is gone -- styles, scripts, the blob, the input field, all of it. The decrypted app loads like it was always there. view-source still shows the encrypted version because browsers cache the original response. DevTools shows the live DOM but that only lives in your tab. Close it and it's gone.

## Crawl protection

```
# robots.txt
User-agent: *
Disallow: /
```

```
<meta name="robots" content="noindex, nofollow, noarchive, nosnippet">
```

Nothing for bots to find.

## Projects

```
PROJECT       DESCRIPTION                STATUS
---------     ---------------------      -----------
Riptide       Signal Architecture        Bespoke
Apex          Sovereign AI+              Active
Castnet       Cognitive Meta-Analysis    Invite Only
```

## Files

```
.
├── index.html      # gate + encrypted payload (~145K)
├── favicon.svg     # icon
├── logo.png        # logo
├── robots.txt      # disallow all
├── CNAME           # www.floworca.com
└── .gitignore      # decrypted.html, preview.html, crypto.mjs
```

## Version history

Every commit re-encrypts the entire blob with a fresh salt and IV. Git sees a full replacement each time. That's by design -- each commit is a complete encrypted snapshot of the site. Roll back to any commit and you get a working, decryptable version from that point in time. Git history doubles as versioned backup.

## The idea

Keep the work locked down until it's ready. No cloud auth, no tokens, no third parties.

One file, one key, one way in.
