# Decisions log

One entry per real design choice, newest at the bottom. Write each in your own words.

Entry format:

## NNN. Short title (date)

- **Decision:** what you chose
- **Alternatives rejected:** what else you considered
- **Why:** the reasons, including what you give up by choosing this
- **Revisit if:** what would make you change your mind

---

## 1. Stack (11/16/26)

- **Decision:** TypeScript and Fastify
- **Alternatives rejected:** Python and FastAPI. I didn't evaluate Python deeply. It was rejected simply because TypeScript fit for me and I don't have the experience to compare further.
- **Why:** TypeScript is closer to Swift, especially WRT static typing. My small amount of backend code for FourSix is also written with TypeScript already, though that was mostly written by Claude, so this would be more about being able to write it myself. But it means I can go back and better understand the code that Claude wrote for me and maintain it better going forward. What I give up is being able to write and understand Python, which feels like such a common foundational language in programming, like something I would have learned if I studied Computer Science. I don't know if there is any value in that though, and it's difficult to determine that value without knowing things I don't know. The foundational concepts are more important here though, and should transfer to any stack or language.
- **Revisit if:** I start seeing interesting job postings that cite the need for Python and nothing about TypeScript. I discover some alternative use case for Python that makes me want to learn it that I am currently unaware of.

## 2. 404 vs 403 response for requests for another user's team (11/16/26)

- **Decision:** 404
- **Alternatives rejected:** 403 would tell the client that they are trying to access something they are not authorized to access.
- **Why:** 404 hides from the client that another user's team exists at the given location. It maintains privacy and security by obfuscating information that the client/user should not be able to learn and use maliciously. If we reveal it's a valid location for another user's data, then they can use that information to try targeted attacks. This only works if we treat "doesn't exist" and "not yours" the exact same way. However, we do give up having more accurate information, which could be helpful for debugging or accurate error messaging for the client to have. But this project I control both client and server, so I have access to logs from both and could see the real reason if needed. This would just help enforce not revealing information to a user that they don't need. It also doesn't materially change much. The user is logged in (otherwise we'd return a 401). At best, a legit user may have multiple accounts for some reason and we could tell them they're trying to access something with the wrong account. But that doesn't seem worth the alternative security implications.
- **Revisit if:** Maybe publically shared teams would change the approach? I have not thought that possibility through, but we can revisit if we get there and discover 403 makes sense for things that are intended to be public and not hidden. E.g. public teams would be accessible via `GET` for everyone but not `PUT`. A 403 for `PUT` in that case seems plausible. But only if publically shared. This would likely be a toggleable setting, and so maybe we'd have to modify the response based on whether something is set to be public or not.