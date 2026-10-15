# The law behind fan games: what renaming and "no official art" do, and what they don't

*Research brief for **Algebraic!** (GDD §5). Compiled 2026-10-12. Written to answer the user's question: "How do Jujutsu Shenanigans, Blox Fruits and all the anime games get away with it? I think they just change the names and avoid using official art."*

> **This is not legal advice.** It is a research brief to help you frame questions for entertainment/IP counsel, who should review the final plan before M0 (GDD §21.2).

**Tag key.**
- **[Docs]**: I read Roblox's official creator-docs snapshot (2026-10-10) directly.
- **[Web]**: a web source, with its URL. WebFetch was DNS-blocked again this session (automaton-media.com), so every [Web] claim rests on **search-engine result snippets**, not on full opinions or articles.
- **[Memory]**: my prior knowledge, *not re-verified* this session. Case citations marked [Memory] should be checked by counsel before anyone relies on them.
- **[Inference]**: my own analysis.

**Budget:** 24 web searches. I did not clone any repos.

**Companion note:** `research2/roblox-cases.md`, a sibling researcher's case table of Roblox fan-game enforcement events. This brief covers the *law*; that note covers the *history*.

---

## 0. Bottom line (read this if nothing else)

**1. Mostly, they don't get away with it *legally*. They get away with it *practically*.** No US doctrine makes a monetized fan game legal because it renamed its characters or redrew them. These games survive because the rights holders have **chosen not to act so far**. Only the rights holder can act. [Inference, grounded in §2–§6]

**2. Renaming helps with trademark and with being found. It does nothing for copyright.**
- **Trademark protects names, titles and logos** as signs of *who made or approved a product*. Calling a character "Big Green Guy" instead of "Hulk" can defeat a trademark claim. That is exactly what happened in *Disney v. Characters for Hire* (S.D.N.Y. 2018). In the same case, **Disney's copyright claims against the same renamed characters survived** [Web: https://www.loeb.com/en/insights/publications/2018/08/disney-enterprises-inc-v-characters-for-hire-llc].
- **Renaming also keeps a game out of brand searches** and away from Roblox's keyword-based IP scanning, which starts from the rights holder's registered mark. [Docs: `ip-licensing/get-started.md`]
- **Copyright protects the character itself:** its look, personality and distinctive traits, *whatever it is called*. [Web/Memory, §1]

**3. Avoiding official art removes the *easiest* claim, not the claim.**
- Ripping official models, frames or audio is literal copying, which is trivial to prove. Re-drawing or re-modelling a character is still copying the character.
- In *DC Comics v. Towle* the defendant built his own Batmobile replicas and still infringed [Web: https://en.wikipedia.org/wiki/DC_Comics_v._Mark_Towle].
- In *Warner Bros. v. X One X*, new composites and 3-D products drawn from film characters infringed, even though the source stills were public domain [Web: https://www.loeb.com/en/insights/publications/2011/07/warner-bros-entertainment-et-al-v-x-one-x-produc__].
- In December 2025, the *Jujutsu Kaisen* production committee publicly called out mobile games with *renamed* but "oddly similar" character designs as copyright infringement and promised takedowns [Web: https://respawn.outlookindia.com/pop-culture/pop-culture-news/jujutsu-kaisen-team-moves-against-unlicensed-mobile-games].

**4. The successful anime games borrow less than people think, and in different places.**
- *The Strongest Battlegrounds* and, to my knowledge, JJS bolt **movesets and VFX onto the player's own Roblox avatar** rather than shipping full character models [Memory]. Blox Fruits' core borrowing is an *idea*: eating a fruit gives you a power.
- JJS's title uses "Jujutsu", an ordinary word for a martial art, not the full mark "Jujutsu Kaisen". [Inference]
- **Ideas, mechanics and generic power fantasies are free to copy. Specific characters are not.** (§2)

**5. Algebraic! as designed is the opposite profile.**
- **The design is likeness-first:** canon character models, Variants, catchphrase barks, Tree Fort and Candy Kingdom recreations, "a fan should recognize the episode a move comes from" (GDD §3.3 P2).
- **The rights holder is the most likely kind to enforce:** a US studio with a kids' brand, first-party Roblox games and a franchise in active production (GDD §5.1).
- **For copyright, the more recognizable the game is, the more it infringes.** A codename version of *this* design would be high risk. [Inference]

**6. Since 2025, the likely outcome is a bill as well as a takedown.**
- Once a rights holder registers on Roblox, it gets **daily automatic matches** of games using its IP, **bulk removal requests (up to 250 links each)**, and **license offers that auto-activate after 14 days unless disputed**, at a revenue share it picks (0–95%). [Docs: `ip-licensing/license-manager.md`, `ip-licensing/creators.md`, `production/publishing/rights-manager.md`]
- **Enforcement on Roblox is now cheap.** The old "it costs too much to chase fan games" defense is weakening. [Inference]

**What this means for the plan.** The GDD's recommendation stands: license-ready, IP-separable, with a Track B original-IP fallback. This brief adds a concrete **"recognizability test"** and an **element-by-element risk matrix** (§9) that the Studio agent can apply as build rules (§11).

---

## 1. Copyright in characters (US)

### 1.1 What copyright protects, in plain English

**Copyright protects *expression*, not *ideas*.** The statute says protection does not extend to any "idea, procedure, process, system, method of operation, concept, principle, or discovery" (17 U.S.C. §102(b)). [Memory; Roblox's own IP page says the same in plain words: "Copyright does not protect facts or ideas, but it may protect the specific expression of a fact or idea." Docs: `production/publishing/ip-guidelines.md`]

- **Idea (free):** a boy hero and his shape-shifting dog in a candy-coated post-apocalypse.
- **Expression (protected):** *Finn*, a specific design (white bear-ear hat, blue shirt, green backpack), voice, personality and history. *Jake*, a specific yellow-orange bulldog with jowls and specific stretchy gags, plus their specific relationship and episodes.

**Characters are protected as part of the works they appear in.** Adventure Time's characters are protected through the copyrighted episodes, comics and art. A separate "character registration" is not needed. [Memory]

**Names and short phrases are *not* protected by copyright.** That covers names, titles, slogans and catchphrases (37 C.F.R. §202.1(a)). [Memory] **This is why many people think renaming is enough:** a name *alone* was never the copyright problem. Names are a **trademark** issue (§3).

### 1.2 When is a character protectable? The tests

| Case | Holding / test | Why it matters for us |
|---|---|---|
| ***Nichols v. Universal Pictures*** (2d Cir. 1930, Learned Hand) [Memory] | "The less developed the characters, the less they can be copyrighted." **Stock characters are free. Well-developed characters are protected.** | Generic archetypes ("sword hero", "ice wizard", "vampire queen") are free. Adventure Time's main cast, developed over 10+ seasons, is not. [Inference] |
| ***DC Comics v. Towle*** (9th Cir. 2015), 802 F.3d 1012 [Web: https://en.wikipedia.org/wiki/DC_Comics_v._Mark_Towle; https://www.loeb.com/en/insights/publications/2015/10/dc-comics-v-towle-dba-garage-gotham] | **Three-part test:** a character is protected if it (1) has **physical as well as conceptual qualities**; (2) is **sufficiently delineated** to be *recognizable as the same character across appearances*, which does not require an identical look every time; and (3) is **especially distinctive**, with "unique elements of expression." The Batmobile passed. Its different looks across versions were treated as "costume changes." | Towle **built his own replicas and used no DC art**, yet still infringed. "Costume changes" don't break identity. **A redrawn, renamed or re-styled Finn is still Finn** if he's recognizable. [Inference] |
| ***Anderson v. Stallone*** (C.D. Cal. 1989) [Memory] | A fan-written *Rocky IV* treatment that reused the Rocky characters was an **unauthorized derivative work**. The characters were so highly delineated that they were protected. | **Fan works that reuse famous characters infringe even when the plot is new.** The infringer also gets no protection for the parts that unlawfully use the characters. [Memory] |
| ***Warner Bros. v. X One X Productions*** (8th Cir. 2011), 644 F.3d 584; damages affirmed 2016 [Web: https://www.loeb.com/en/insights/publications/2011/07/warner-bros-entertainment-et-al-v-x-one-x-produc__; https://www.jdsupra.com/legalnews/surrender-dorothy-court-upholds-damages-18612/] | Reproducing a single *public-domain* publicity still was allowed. **Juxtaposing images, adding phrases, or extending an image into 3-D infringed** the copyrighted film characters (*Wizard of Oz*, *Gone with the Wind*). On remand: **$10,000 × 257 items = $2.57M** in statutory damages, affirmed. | **Turning 2-D characters into 3-D models and new scenes is exactly the kind of "new arrangement" that infringed.** A Roblox model of a cartoon character is a 3-D extension. [Inference] |
| ***MGM v. American Honda*** (C.D. Cal. 1995) [Memory] | An unnamed, Bond-like hero in a Honda ad (tuxedo, quips, villain, helicopter chase) was **likely infringing** on the James Bond character. A preliminary injunction issued. | **No name needed.** A recognizable *combination* of traits, look and situations can be enough. [Memory] |
| ***Daniels v. Walt Disney Co.*** (9th Cir. 2020) [Memory] | The "Moodsters" (color-coded emotion characters) were **not** sufficiently delineated, so there was no protection. | **The limit:** thin, generic characters lose. This helps *us* in Track B, where "a guy who controls ice" is free. [Inference] |
| ***Disney v. Sarelli (Characters for Hire)*** (S.D.N.Y. 2018), 322 F. Supp. 3d 413 [Web: https://www.loeb.com/en/insights/publications/2018/08/disney-enterprises-inc-v-characters-for-hire-llc; https://archive.techdirt.com/articles/20180814/09370240427/disney-gets-early-loss-trademark-copyright-suit-against-unlicensed-birthday-party-characters.shtml] | A party company dressed performers as Disney/Marvel/Star Wars characters under **generic names** ("Big Green Guy", "The Dark Lord"). **Trademark infringement failed (no likely confusion). Dilution survived. Copyright survived**, though summary judgment was denied to both sides on evidence grounds. The descriptive and nominative fair-use defenses were rejected. Later history not verified. | **This is the closest real-world test of the "just rename them" theory.** Renaming beat the trademark claim, not the copyright claim. |
| ***Castle Rock v. Carol Publishing*** (2d Cir. 1998) [Memory] | A *Seinfeld* trivia book infringed. **"Facts" about a fictional world are the author's expression, not facts.** | **Lore cards, quizzes and trivia about Ooo are not "just facts".** (§9: lore text) |
| ***Micro Star v. FormGen*** (9th Cir. 1998) [Memory] | Selling user-made *Duke Nukem 3D* levels infringed. The levels were **derivative works, effectively unauthorized sequels** set in the game's world. | **Selling new adventures set in someone else's world is a derivative work, even if all the assets are your own.** |
| ***Salinger v. Colting*** (2d Cir. 2010) [Memory] | An unauthorized "sequel" using Holden Caulfield was likely infringing. The fair-use defense was rejected at the preliminary stage. | **Fan sequels are a classic loser on fair use** (§4). |

### 1.3 How infringement is actually decided: copying + substantial similarity

To win, the owner must show (a) it **owns a valid copyright** and (b) the defendant **copied protected expression**. Copying is usually shown by *access + similarity*. [Memory]

- **"Substantial similarity" is measured on the *protected* parts only.** Courts filter out ideas, stock elements and *scènes à faire* first, then ask whether an *ordinary observer* would see the protected expression as taken. The Ninth Circuit calls these steps the "extrinsic" and "intrinsic" tests. [Memory]
- **For characters, that question collapses into one practical test: would an ordinary fan recognize *that specific character*?** If yes, you have a strong copying signal. Towle's "recognizable as the same character" language points the same way. [Inference]

**Access is a given for Adventure Time.** It is famous, and our own GDD cites episodes and art books. *Our own design documents are evidence of copying and intent.* [Inference] See §11 rule 7.

### 1.4 Derivative works: the right we would actually infringe

- **The owner has the exclusive right to "prepare derivative works"** (17 U.S.C. §106(2)). A derivative work is a work "based upon" the original: a translation, dramatization, "or any other form in which a work may be recast, transformed, or adapted" (§101). [Memory]
- **A video game adaptation of a cartoon is a textbook derivative work.** It is exactly what Cartoon Network licenses to game makers. [Inference]
- **Unauthorized derivative works earn no protection** for the parts that unlawfully use the original (§103(a)). This is why an unlicensed fan game has weak legal standing against *its own* copycats for IP-bearing content. Original code and original assets are still its own. Example: Gamer Robot (Blox Fruits) did get a copy game, "Blox Fruits: Recreation", removed on a copyright claim [Web: https://progameguides.com/roblox/what-happened-to-blox-fruits-recreation/]. [Memory + Inference]

### 1.5 So what does "renaming + no official art" do, element by element?

| Change | What it fixes | What it does **not** fix |
|---|---|---|
| **Rename characters** ("Finn" → "The Last Human") | Most **trademark** confusion (*Characters for Hire*) [Web]. **Keyword-based detection** by Roblox's License Manager and by brand searches [Docs]. Lowers the "this is official" impression. | **Copyright in the character's look, personality and story role.** The *combination* still identifies the character (*MGM v. Honda*). Epithets that quote canon ("The Last Human", "Vampire Queen") can *strengthen* the identification. [Inference] |
| **Don't use official art/audio** (no ripped models, frames or clips) | The **easiest-to-prove claim:** literal copying, which needs no similarity debate. **Japan's ex officio criminal path**, which (per my understanding) targets "as-is" copies for profit (§6) [Memory]. Ripped-asset scanning hits. | **Derivative-work and character infringement.** Re-modelled, re-drawn or 3-D re-creations still copy expression (*Towle*, *X One X*). [Web] |
| **Use generic power names** ("Flame", "Ice", "Stretch") | Nothing was protectable in these words anyway. | Nothing to fix, *if* the visual realization is also generic. [Inference] |
| **Put movesets on the player's own avatar** (instead of a character model) | **Removes the single most protected element:** the character's appearance. What remains is moves, VFX and naming. [Inference] | **Signature moves that reproduce specific, distinctive scenes** can still be expression (§2). The thinner the copying, the lower the risk, but it is not zero. [Inference] |

---

## 2. Game mechanics: free as ideas, protected as expression

### 2.1 The rule

**Rules, systems and methods of play are §102(b) ideas or "methods of operation". Anyone can copy them.** [Memory]
- *4-hit M1 chain, block, perfect block, dash, awakening meter, sure-hit "domain" zone, gacha with pity*: all free.

**The *expression* of a game is protected:** its art, characters, audiovisual sequences, text and music, and sometimes the specific look of its pieces and board. [Memory/Web below]

### 2.2 The game cases

| Case | What happened | Lesson |
|---|---|---|
| ***Data East v. Epyx*** (9th Cir. 1988) [Memory] | A karate game's similarities to another (moves, scoring, referee) were **unprotectable**. They are inherent in the idea of a karate game (*scènes à faire*). | **Generic fighting moves are free.** |
| ***Capcom v. Data East*** (N.D. Cal. 1994): *Street Fighter II* v. *Fighter's History* [Web: https://en.wikipedia.org/wiki/Capcom_U.S.A._Inc._v._Data_East_Corp.; https://patentarcade.com/2005/08/case-capcom-v-data-east-nd-cal-1994-c.html] | Capcom alleged copying of characters, special moves and control sequences. The preliminary injunction was **denied**. Reporting says the court found many moves were generic martial-arts moves, and that what was copied was unprotectable. The final disposition date is unconfirmed. | **The most on-point precedent for our genre.** A fighting game's *special moves as moves* are mostly unprotectable. Copying a *specific character's distinctive look* is a different question. |
| ***Atari v. North American Philips*** (7th Cir. 1982), Pac-Man v. *K.C. Munchkin* [Memory] | Despite different mazes and details, *K.C. Munchkin* **infringed**. The **gobbler character and ghost "monsters"** captured Pac-Man's expressive look and feel. | **Characters in games are the vulnerable layer, even when the mechanics are free.** |
| ***Tetris Holding v. Xio Interactive*** (D.N.J. 2012), 863 F. Supp. 2d 394 [Web: https://www.loeb.com/en/insights/publications/2012/06/tetris-holding-llc-v-xio-interactive-inc; https://newmedialaw.proskauer.com/2012/06/19/videogame-app-developer-breaks-the-rules-on-copyright-infringement/] | *Mino* copied Tetris's rules (lawful) **and** its look: piece style, colors, movement, board dimensions. **Copyright and trade dress were infringed.** The court said the merger and *scènes à faire* defenses defined "function" too broadly. There were countless other ways to *express* the same rules. Appellate history not verified. | **"It's just the mechanics" fails if you also copy the audiovisual expression.** Courts look at the overall look and feel. |
| ***Spry Fox v. 6waves (LOLApps)*** (W.D. Wash. 2012) [Web: https://www.pocketgamer.com/news/6waves-settles-with-spry-fox-in-yeti-town-triple-town-clone-debate/; https://mcvuk.com/development-news/triple-town-cloning-suit-settled/] | *Yeti Town* re-skinned *Triple Town* (yetis for bears). The court **denied 6waves' motion to dismiss**, rejecting the "copyright doesn't protect against a game-design clone" defense. The case **settled**: Spry Fox got the *Yeti Town* IP and the game was pulled. **No merits ruling.** | **A re-skin of a distinctive game can be pled as infringement and survive.** Don't over-read it as a final ruling. |
| ***DaVinci Editrice v. ZiKo Games*** (**S.D. Tex.** 2016), 183 F. Supp. 3d 820 [Web: https://copyrightalliance.org/copyrightability-of-board-games/; https://techraptor.net/content/game-mechanics-arent-protected-copyright-according-texas-court-ruling] | *Legend of the Three Kingdoms* copied *Bang!*'s mechanics almost exactly but re-themed the art and setting. **No infringement.** The court held that **a character's *special abilities* are "a subset of the rules,"** not literary or artistic expression. | **The strongest authority for "same moveset, different character" being lawful.** *Correction to the task brief:* this is a **US (Texas)** decision, not Australian. I found no Australian *DaVinci v. Ziko* case. |

### 2.3 "A stretchy-dog moveset" vs. "a dog named Jake that looks like Jake"

| Design | Analysis | Risk |
|---|---|---|
| **A "Stretchy Companion" moveset** on the player's own avatar or on an original rubbery capybara: elongate-punch, giant-fist slam, body-as-shield, grow-big awakening | **Idea and mechanics.** Stretchy fighters are a stock trope (Plastic Man, Mr. Fantastic, Luffy, Jake). The *DaVinci* reasoning treats character abilities as rules. *Capcom v. Data East* treats generic special moves as free. [Web + Inference] | **Low** |
| **The same moveset, with VFX and animations that restage specific, distinctive Jake gags** from named episodes (e.g., turning into a specific object, a key set-piece from an episode) | **Thin but real expression:** specific audiovisual sequences from specific works. Each one alone may be minor. Together, plus GDD P2 ("a fan should recognize the episode"), they build a substantial-similarity case. [Inference] | **Medium** |
| **A yellow-orange bulldog with jowls and a buddy-dog personality, paired with a boy in a white bear hat, named "Magic Dog"** | **Recognizable character = copying protected expression.** The renamed epithet is irrelevant (*Towle*, *MGM v. Honda*, *Characters for Hire* copyright claim). [Web/Memory] | **High** |
| **"Jake", looks like Jake, sounds like Jake** | **Copyright + trademark + possibly the voice actor's right of publicity** (§3, §9). | **Critical without a license** |

---

## 3. Trademark: names, titles, logos, catchphrases

### 3.1 What trademark protects

**A trademark is a source identifier:** a word, logo, design or even sound that tells consumers *who stands behind* a product. [Docs: `production/publishing/ip-guidelines.md`; `ip-licensing/faq.md` gives "Stranger Things" and "Polly Pocket" as examples]

**Main claims** [Memory]:
- **Infringement of a registered mark:** Lanham Act §32, 15 U.S.C. §1114.
- **False designation of origin / false endorsement, registered or not:** §43(a), §1125(a).
- **Dilution of a *famous* mark:** §43(c).

**Core test: likelihood of confusion** about source, sponsorship or affiliation. It is a multi-factor test (*Polaroid* in the 2d Cir., *Sleekcraft* in the 9th Cir.). The factors include strength of the mark, similarity, relatedness of goods, evidence of actual confusion, **the defendant's intent**, and the marketing channels. [Memory]

### 3.2 Why descriptive epithets reduce trademark exposure

- **Epithets are ordinary words used in their ordinary sense.** "Vessel", "Honored One", "Green Hero" are not anyone's registered marks. They don't signal "this is the official product". So **one of the main confusion levers disappears.** [Inference] *Characters for Hire* won on exactly this point. [Web]
- **Statutory "descriptive fair use"** (§33(b)(4), 15 U.S.C. §1115(b)(4)) protects using a term "otherwise than as a mark… in good faith only to describe" one's goods [Memory]. So "Jujutsu" (a martial art), "adventure", "algebraic" and "ice" can be used *descriptively*.
- **But epithets do nothing for copyright** (§1). Epithets that *quote* canon titles ("The Vampire Queen", "The Last Human", "Ice King") may *help* a plaintiff show the character was copied on purpose. [Inference]

### 3.3 Nominative fair use and the *Rogers* test: what they allow

**Nominative fair use** (*New Kids on the Block v. News America*, 9th Cir. 1992) [Memory]: you may use *another's mark to refer to their product* when (1) the product isn't readily identifiable without it, (2) you use only as much as needed, and (3) you suggest no sponsorship.
- **Fine:** "*Algebraic!* is not affiliated with Cartoon Network" in a disclaimer. [Inference]
- **Not fine:** "Adventure Time battlegrounds" in the title, tags or thumbnail. That uses the mark *to attract players to your product*, and it implies affiliation. [Inference]
- *Characters for Hire* rejected the nominative fair-use defense for the costumed-character business. [Web]

***Rogers v. Grimaldi*** (2d Cir. 1989) used to shield artistic works' titles. ***Jack Daniel's v. VIP Products*** (U.S. 2023) held that *Rogers* **does not apply when the mark is used as a source identifier for the defendant's own goods.** A game title that uses someone else's mark is the classic case. [Memory]

**Disclaimers help at the margin but do not cure a confusing title, and do nothing for copyright.** [Memory/Inference]

### 3.4 Are "Adventure Time", "Finn", "Jake", "Ooo" registered marks?

| Term | What I found | Confidence |
|---|---|---|
| **ADVENTURE TIME** | **Multiple USPTO filings owned by The Cartoon Network, Inc.** <br>- Serial **85477828** (filed 6 Dec 2011): *"audio and video recordings… interactive video game programs; video game cartridges…"* (**games are covered**).<br>- **77619257** (filed 21 Nov 2008): entertainment services, a multimedia program series.<br>- **86136816** (2014): pet products.<br>- **86556009** (2015) and **97844713** (2023): bottles and containers.<br>**Current status of each: not verified.** [Web: https://trademark.justia.com/owners/the-cartoon-network-inc-2453528/page6; https://trademark.justia.com/owners/the-cartoon-network-inc-796228/page11] | Medium (aggregator data) |
| **ADVENTURE TIME WITH FINN & JAKE** | Filing **77637612** (Cartoon Network). Goods and status not visible. [Web: Justia owner listing, same URLs] | Low–Medium |
| **FINN, JAKE, BMO, MARCELINE, ICE KING, OOO / LAND OF OOO** | **No standalone registrations surfaced** in two searches. **Absence in search ≠ no registration.** "Finn" and "Jake" are common given names and would be weak standalone marks. "BMO", "Marceline", "Lumpy Space Princess" and "Princess Bubblegum" are more distinctive. Unregistered marks can still be enforced under §43(a) if they function as source identifiers (e.g., on licensed merchandise). [Web + Memory + Inference] | Low (unverified) |
| **Who handles enforcement** | A related Cartoon Network mark lists **Warner Bros. Discovery Legal TM Group (Atlanta)** as correspondent. [Web: https://www.trademarkelite.com/trademark/trademark-detail/87097039/APOLLO-GAUNTLET] | Medium |
| **ALGEBRAIC / MATHEMATICAL / "WHAT TIME IS IT?"** | **Not searched.** These are show catchphrases. "Algebraic!" is **our game title**. → **Open question for counsel: run a full clearance search.** | n/a |

**Takeaway.** "Adventure Time" in a game title, description, tags or thumbnail is **the single clearest trademark problem**. There is an application or registration on file that covers video games. Character names are a second-tier trademark issue, but a first-tier *detection* issue (see the License Manager keywords below). [Inference]

### 3.5 Trademark-adjacent: how Roblox's IP scanning uses names

**How matching works:** a registered rights holder builds an "IP family" with a **primary keyword** ("the official registered mark of your IP"), **secondary keywords** (characters, branded elements, fictional places) and **media** (at least 10 approved images are needed for matching). [Docs: `ip-licensing/get-started.md`, `license-manager.md`]
- "Roblox uses the primary and secondary keywords and the media added to your IP family to find creations that include content that matches your IP." **Matches refresh daily** and cover public, actively played games. [Docs: `ip-licensing/license-manager.md`]
- **Implication:** renaming lowers *keyword* matches, but **image matching still applies** to characters that look like the source. **Being hard to find is not the same as being legal**, and image matching makes it a weaker shield over time. [Inference]

**Discovery cost:** Roblox's discovery FAQ lists **"misleading or mismatched metadata and content"** as a reason impressions can fall. [Docs: `discovery-faq.md`] Stuffing someone else's brand into your tags is risky for both legal and discovery reasons. [Inference]

---

## 4. Fair use and parody: why a commercial fan game almost never qualifies

**Fair use (17 U.S.C. §107) weighs four factors** [Memory]:

| Factor | A monetized Roblox fan game |
|---|---|
| **1. Purpose and character.** Is the use commercial? Is it *transformative*, adding new meaning or a different purpose? | **Commercial:** Robux passes, Wishes, private servers. **Same purpose** as the original: entertaining fans with these characters. *Warhol Found. v. Goldsmith* (U.S. 2023) narrowed "transformative" when the use **shares the original's purpose and is commercial**. [Memory] **Weighs against.** |
| **2. Nature of the work** | **Highly creative fiction**, the core of copyright. **Against.** |
| **3. Amount and substantiality** | **The main characters are the "heart" of the work.** A roster of 18 canon characters takes a lot. **Against.** |
| **4. Effect on the market**, including the market for *licensed derivatives* | **Licensed game adaptations are a real market.** Cartoon Network already runs first-party Roblox games (GDD §5.1). Roblox's License Manager now makes the Roblox licensing market concrete. An unlicensed game substitutes for a license. **Strongly against.** [Inference] |

**Parody** (*Campbell v. Acuff-Rose*, U.S. 1994) favors works that **comment on or criticize the original**. A brawler that *uses* the characters to fight, rather than *mocking or commenting on* the show, is not parody in the legal sense. Calling it "shenanigans" doesn't change that. [Memory + Inference]

**Myths Roblox itself debunks.** Roblox's IP page says you **can still infringe even if you**:
- didn't monetize;
- credited the original creator;
- saw others do the same thing;
- thought the use was fair use.

[Docs: `production/publishing/ip-guidelines.md`]

**Verdict:** *fair use is not a realistic defense for any monetized track-C version of Algebraic!* [Inference]

---

## 5. Enforcement mechanics: who acts, how, and with what consequences

### 5.1 Only the rights holder (or its agent or exclusive licensee) can act

- **Copyright and trademark are private rights.** The DMCA notice must be signed by the owner or an authorized agent, under penalty of perjury. [Docs: `ip-guidelines.md`, citing 17 U.S.C. §512(c)(3)] **Third parties, rival developers and fans cannot validly DMCA your game for someone else's IP.** [Inference from the notice requirements]
- **Roblox's IP policy itself "prohibits the unauthorized use of intellectual property, and removes… content that violates this policy."** It suspends or terminates repeat or egregious infringers. [Docs: `ip-guidelines.md`]
- **In practice Roblox's removals are notice-driven.** The sibling note found no evidence that Roblox proactively polices anime fan games. Roblox's licensing FAQ still tells creators to *"remove any content that might use IP and isn't covered by a license."* [Docs: `ip-licensing/faq.md`; Web via `roblox-cases.md`]

### 5.2 The DMCA notice / counter-notice cycle on Roblox

1. **Notice.** The rights holder sends a §512(c)(3) notice (email, mail, or the **Rights Manager** IP Platform, up to **250 links per request**). [Docs: `ip-guidelines.md`, `rights-manager.md`]
2. **Takedown.** Roblox "will act quickly" to remove the content and notify the creator. [Docs]
3. **Counter-notice** (§512(g)(3)). The creator swears, under penalty of perjury, that removal was a *mistake or misidentification*. The creator also **consents to federal court jurisdiction** and accepts service of process. [Docs]
4. **Restoration.** Roblox "generally restore[s] the content within 14 business days unless the reporter provides information showing that they have filed a lawsuit." The statute's window is 10–14 business days (§512(g)(2)(C)). [Docs + Memory]
5. **Abuse.** Knowingly false notices or counter-notices create liability under §512(f). [Docs]

**Trademark notices:** there is no statutory counter-notice. The creator appeals to Roblox by email. [Docs] **EU:** a parallel Digital Services Act notice and appeal path exists, with a six-month appeal window. [Docs: `dsa-ip-reporting.md`]

**Why counter-noticing an actually infringing fan game is a bad idea.** It **invites a lawsuit**: the rights holder only has to file suit to keep the game down, and you've already consented to jurisdiction. It may also be a §512(f) misrepresentation. [Inference]

**Why Roblox complies.** §512's **safe harbor** shields platforms from damages for user-uploaded infringement *only if* they remove content expeditiously on valid notice and keep a repeat-infringer policy (§512(c), §512(i)). **Roblox has every incentive to remove first.** [Memory]

### 5.3 Roblox-specific (2025–26): License Manager turns "takedown" into "tax or takedown"

- **Offers.** A registered rights holder can send a **license offer** to any matched public game. A **standard offer auto-activates after 14 days unless you dispute it.** [Docs: `ip-licensing/creators.md`]
- **Disputes.** You may dispute twice: "does not use this IP", "fair use", or "I removed the IP". If you dispute the second time, the offer is archived. The rights holder "might choose to pursue an IP infringement claim separately." [Docs]
- **Conditional offers.** These require you to change content to meet the holder's standards within 14 days. They do not auto-activate. [Docs]
- **Rate.** The rights holder sets the revenue share **anywhere from 0% to 95%**, and once used it cannot be changed. [Docs: `license-manager.md`]
- **Early use.** "Early use of IP in your game may lead to early activation of a license request." [Docs: `creators.md`]

**Implication for Track C:** if Cartoon Network/WBD (or a successor owner) registers *Adventure Time*, an unlicensed AT game would most likely get either an offer at a rate the rights holder chooses or a removal request. **Neither outcome is in our control.** [Inference]

### 5.4 Remedies if it goes to court (US)

| Remedy | Detail |
|---|---|
| **Injunction** | Courts can order the infringing work taken down and kept down. [Memory] |
| **Actual damages + infringer's profits** (§504(b)) | Profits *attributable to* the infringement. For a monetized game this could reach DevEx revenue. [Memory + Inference] |
| **Statutory damages** (§504(c)) | **$750–$30,000 per work infringed**, up to **$150,000 per work if willful**, down to $200 if innocent. Available only if the work was registered in time (§412). Big studios register routinely. [Memory] Real example: *X One X*, **$2.57M** [Web]. |
| **Attorney's fees** (§505) | Discretionary, and also tied to timely registration. [Memory] |
| **Trademark** | Defendant's profits, damages, costs, possibly trebled. Injunction. [Memory] |
| **Willfulness evidence** | Design docs that say "make it recognizable as Jake but call it Magic Dog" are discoverable and **prove intent.** [Inference] |

**Cease-and-desist letters are the usual first step** for fan projects. They demand you stop, take content down and sometimes account for revenue. Most fan cases end there.
- **Lawsuits tend to target *commercial* operations:** Towle sold replica cars; *X One X* sold merchandise; *Characters for Hire* ran a party business; *Axanar* raised over $1M (see §6.3). [Web/Memory + Inference]
- **A monetized Roblox game is a commercial operation.** [Inference]

### 5.5 Why so many infringing works persist

| Reason | Explanation |
|---|---|
| **Enforcement is discretionary** | Nothing forces a rights holder to sue. **Copyright doesn't "lapse" from non-enforcement** the way people sometimes think. Trademark owners do have some incentive to police, but tolerating fan works rarely forfeits a famous mark. [Memory] |
| **Cost vs. value** | Small games weren't worth legal hours. **Roblox's 2025 tools (daily auto-matching, 250-link bulk removals) collapsed that cost.** [Docs + Inference] |
| **PR risk** | Suing kids' fan creations looks bad. WB's 2018 Harry Potter festival letters drew public criticism. [Web: https://slashfilm.com/harry-potter-fan-festivals-crackdown-warner-bros] |
| **Free marketing** | Fan games keep a franchise visible to a young audience. Many owners quietly accept that. [Inference] |
| **Fragmented ownership** | Anime rights are often split across production committees, publishers and overseas licensors, so someone must coordinate a claim. [Memory] The one clear JJK-family Roblox enforcement went through **Viz Media**, Shueisha's US arm (*Shinobi Life 2*, Naruto IP). [Web via `roblox-cases.md`] |
| **No competing product (yet)** | Enforcement spikes when the rights holder or its licensee launches its own Roblox game. Example: the alleged late-2023 claims against *Anime Adventures* and *Anime Fighting Simulator X*, reportedly tied to Gamefam's licensed *My Hero Academia* partnership. **Unconfirmed.** [Web: https://www.dualshockers.com/roblox-anime-adventures-removed/; https://www.pockettactics.com/anime-adventures/private; https://x.com/Roblox_RTC/status/1738005940128100735] |
| **Survivorship bias** | We remember the games still up, like *Anime Battle Arena* after 8+ years [Web: https://abawiki.wiki/resources/roblox-copyright/]. We forget *Pokémon Brick Bronze* (2018) and the 2016–2021 Nintendo fan-game purges: 550+ games on Game Jolt in 2016, 379 in 2021 [Web: https://gamesbeat.com/nintendo-allegedly-files-dmca-takedown-notice-over-more-than-550-fans-games/; https://www.nintendolife.com/news/2021/01/nintendo_issues_mass_dmca_takedown_379_fan-made_games_forcibly_removed]. |

---

## 6. Japan's tolerance vs. the US studio posture

### 6.1 Why Japanese rights holders have historically tolerated fan works

**Doujinshi culture.** Fan comics and games sold at events like Comiket are a huge, long-standing scene. They are usually technically infringing, yet "normally tolerated by Japanese publishers". Many professional creators came up through doujin work. [Web: https://Www.wikipedia.org/wiki/Dojinshi (snippet); https://bilaterals.org/japanese-fan-comics-could-die]

**Legal structure.**
- **Criminal copyright infringement in Japan has traditionally required a *complaint* by the rights holder** (a "complaint-based offense", Copyright Act Art. 123). No complaint means no prosecution. [Web: https://bilaterals.org/japanese-fan-comics-could-die; Memory for the article number]
- **The TPP-driven amendment** (in force **30 Dec 2018** with TPP11) *partly* removed the complaint requirement. The Agency for Cultural Affairs explained how this applies to doujin activity. [Web: https://www.itmedia.co.jp/news/spv/1901/17/news105.html]
- **My understanding** [Memory, verify]: the non-complaint path covers only infringement **for profit**, distributing the work **"as is"** (dead copies), and **unjustly harming** the owner's profits. So **parody and derivative fan works generally stay complaint-based.**
- **Civil suits** (injunction, damages) are available either way. [Memory]

**Community tools.**
- The **"Doujin Mark"** (2013, championed by Ken Akamatsu with Creative Commons Japan) lets authors pre-authorize event-sold doujinshi. [Memory]
- Ken Akamatsu warned that banning doujinshi would weaken "the power of the entire manga industry". [Web: https://bilaterals.org/japanese-fan-comics-could-die]

### 6.2 Published Japanese fan-work guidelines (and what they *don't* allow)

| Company / IP | Gist | Source |
|---|---|---|
| **Capcom** (updated Nov 2025) | Fan content allowed within limits. **Commercial use prohibited.** Small-scale, non-business sales appear tolerated. | [Web: https://eventhubs.com/news/2025/nov/07/capcom-fan-content-guidelines] |
| **Square Enix** (NieR, 2021; Towa Tsugai, 2023) | Rules for fan works. Bars conduct that damages the series' image (which confused NieR's own director). Towa Tsugai lets Square Enix showcase fan works. | [Web: https://soranews24.com/2021/02/04/nier-automatas-creator-is-confused-by-square-enixs-new-fan-art-guidelines/amp/; https://automaton-media.com/en/news/20230217-17921/] |
| **Cygames** (Uma Musume) | Content restrictions: no violent, sexual or political use, out of respect for real racehorse owners. | [Web: https://automaton-media.com/en/news/why-does-umamusume-pretty-derby-have-specific-rules-against-sexual-or-violent-fanart/] |
| **Nitroplus** | Fan works OK if not "excessively commercial". Reported caps: **≤200 units cumulative, expected revenue < ¥100,000**, individual creators. | [Web: https://automaton-media.com/en/news/send-us-your-doujinshi-nitroplus-affirms-positive-stance-on-fan-made-content-asks-people-to-show-them-what-theyre-cooking/ (snippet; exact figures from a search summary, not verified on the page)] |
| **Broccoli** (2016) | No direct appropriation of official material. No for-profit sales. Compliant works need no contact. | [Web: https://animenewsnetwork.com/interest/2016-01-09/broccoli-lays-down-the-law-for-dojin/.97335] |

**What the guidelines don't allow** [Inference]:
- **Even the most permissive guidelines exclude corporate-scale, monetized digital products.** A Roblox game with millions of players and Robux monetization falls **outside** every guideline above.
- **JJS-style games are not "permitted" by Japanese guidelines. They are *unenforced*.**
- **Tolerance has limits:** the JJK committee's December 2025 warning about renamed look-alike mobile games [Web, §0], Shueisha's anti-piracy subpoenas [Web: https://torrentfreak.com/?p=234915], and **Nintendo**, a Japanese company with an aggressive fan-game takedown record (§5.5).

### 6.3 Typical US studio posture: rules-based tolerance, or none

**Paramount/CBS *Star Trek* fan-film guidelines (June 2016)** were issued after suing *Axanar*, which settled in Jan 2017. Reported terms [Web: https://www.space.com/35436-star-trek-axanar-fan-film-lawsuit-settled.html; https://trekmovie.com/2017/01/20/axanar-studios-settle-star-trek-copyright-infringement-lawsuit/]:
- non-commercial;
- runtime caps of 15-minute segments, about 30 minutes total;
- **no "Star Trek" in the title**;
- amateurs only;
- fundraising ≤ $50,000.

**Even Paramount's tolerance bans the brand in the title and bans commercial use.** An arbitrator later awarded about $292K in fees over alleged settlement breaches (reported). [Web, same search]

**Warner Bros. (Harry Potter, 2018):** letters to unofficial fan festivals said they **could not use any names, places or objects** from the series, framed as "protecting the trademark". [Web: https://slashfilm.com/harry-potter-fan-festivals-crackdown-warner-bros] **This is the same corporate family as Cartoon Network** (WBD; now reported merged into Paramount/Skydance per GDD §5.1). [Inference]

**Cartoon Network / Adventure Time specifically:** I found **no reported AT fan-game takedown** (one targeted search) [Web: no hits]. **Absence of reports ≠ tolerance policy.** CN's first-party Roblox presence (*Ben 10*, *Game On!*, GDD §5.1) puts it squarely in the "has a product to protect" category. [Inference]

**Contrast in one line:** Japanese owners have historically **tolerated by default and enforced by exception**. US studios **enforce by default and tolerate by published exception**, usually non-commercial only. [Inference]

---

## 7. So how *do* JJS and Blox Fruits get away with it? A synthesis

| Factor | JJS | Blox Fruits | Algebraic! (Track C, hypothetical) |
|---|---|---|---|
| **What is borrowed** | Movesets, VFX and "domain" concepts referencing JJK, applied (to my knowledge) to the **player's avatar**, with epithet names [Memory/GDD §2.1] | Mainly the **idea** of fruit powers and a pirate/marine sea adventure. Generic element names. Renamed from "Blox Piece" in an "Anti-Copyright Update" (Dec 2019) [Web via `roblox-cases.md`: https://rblxguide.com/games/blox-fruits/updates/update-9] | **Canon character likenesses**, canon locations, catchphrases, episode-specific moves, Variants (GDD §7, §9.3, §11) |
| **Title** | "Jujutsu" = an ordinary word, not the full mark [Inference] | Original mark | "Algebraic!" = an AT catchphrase *(clearance unknown)* |
| **Rights holder type** | Japanese publisher/committee, historically tolerant; no Roblox product found [Web via `roblox-cases.md`] | Same | **US studio, kids' brand, first-party Roblox games, active production** (GDD §5.1) |
| **Monetization tied to IP** | Light: cosmetics, early access [GDD §2.1] | Heavy, but tied to *original* fruit items | Wishes for character styles = **paying for the IP itself** |
| **Detectability** | Keyword-light; image-match uncertain | Keyword-light | Keyword-light if renamed, but **image-heavy** |
| **Legal position** | Arguably infringing in parts. Thinner copying than people assume. | Closest to idea-level, so lowest | **Squarely infringing (derivative work + character copying)** |

**The real answer to the user** [Inference]:
1. **Part law:** the most successful games copy at the *idea* level, use player avatars, and avoid brand words.
2. **Part culture:** Japanese owners have tolerated fan works.
3. **Part economics:** enforcement used to be expensive, and fan games are free marketing.
4. **Part survivorship bias.**
5. **Renaming is the least important ingredient.** It's mostly camouflage against keyword search and trademark claims.

---

## 8. A note on the user's "make the cosmetic look optional" change

The user asked to make one cosmetic-look decision a player toggle. I don't know exactly which decision it was, so the following is conditional. [Inference]

- **If the toggle means "the player can keep their own avatar while using a Fighting Style":** that is a good design choice, and it matches the genre norm.
- **It does not lower legal risk if the IP look ships in the game.** Copyright and trademark exposure attach to what we **distribute and sell**, not to what each player happens to select. An infringing character model stays infringing even if it's off by default.
- **Under Track B (original world) the toggle is legally neutral**, because the look is ours.
- **The legally meaningful version** of "avatar-first" is a build where **no IP look is shipped at all.**

---

## 9. Risk matrix: element by element

**Columns:**
- **Track C** = an unlicensed public game with AT content under codenames. The GDD already says ❌ do not launch.
- **Track B** = an original "inspired-by" world (GDD §5.5); the risk shown is what remains.
- **Track A** = licensed. Risk becomes contractual (licensor approval), so it is not scored.

**Risk scale:** **Critical** (likely to produce a takedown or claim on its own), **High**, **Medium**, **Low**.

| # | Element | What the law protects | Track C risk | Track B residual risk | Mitigation / build rule |
|---|---|---|---|---|---|
| 1 | **Game title "Algebraic!"** | **Trademark only** (a single word isn't copyrightable). It is an AT catchphrase; CN registration status unknown. | Medium. Adds evidence of intent when combined with AT content. | **Low–Medium** (pending clearance) | **Counsel clearance search** for ALGEBRAIC (classes 9/41/28) before M0. Have a backup title. Never stylize it like the show logo. |
| 2 | **"Adventure Time", "Cartoon Network", "Ooo"** in title, description, tags, update names, ads, Discord, YouTube | Registered or applied-for marks (video games covered by serial 85477828) [Web]. Also the **primary keyword** for Roblox scanning [Docs]. | **Critical** | **High if used.** Nominative use is fine only in a careful disclaimer, and even that is best avoided. | **Ban these strings** in all public metadata. Optional disclaimer "Not affiliated with…" only if counsel wants it. No "AT battlegrounds" SEO. |
| 3 | **Character names** (Finn, Jake, BMO, Marceline…) | Trademark (some likely protectable) + detection by secondary keywords | High | **Low** (we use original names) | A Track B name list is checked against the canon name list. No near-homophones ("Fin", "Jayk", "Beemo"). |
| 4 | **Epithets** ("The Last Human", "Vampire Queen", "Ice King") | Ordinary words, so low trademark risk. **Canon-quoting epithets are evidence of copying.** | Medium (it's the likeness that matters) | Low | In Track B, use epithets that **don't quote canon titles** ("The Seed Knight", not "The Last Human"). |
| 5 | **Character likeness**: models, silhouettes, color schemes, signature props (bear-ear hat, Jake's jowls/colour, Marceline's axe-bass, Ice King's crown + beard + blue skin, PB's pink + crown, BMO's console body) | **Core copyright** (*Towle*, *Nichols*, *MGM v. Honda*). 3-D re-creations are derivative works (*X One X*). Also possible trade dress. | **Critical** | **Medium.** The risk is *drift* back toward canon. | **Recognizability test** (§10). Change ≥3 of: silhouette, palette, signature prop, species/body type, costume motif. No canon-adjacent pairings (boy + bear hat + yellow dog). Art review sign-off. |
| 6 | **Variants** (Farmworld Finn, *Stakes* Marceline…) | Each is a specific canon design and story element | **Critical** | Low (original variants) | Track B variants are original. No "alt-universe" variants that map 1:1 onto canon arcs. |
| 7 | **Catchphrases** in barks, achievements, UI ("Mathematical!", "Oh my glob!", "What time is it?", "Shmowzow!") | Short phrases aren't copyrightable. Trademark possible if registered or used as a source identifier. Combined with likeness, they are strong evidence of copying. | High (as evidence) / Medium (alone) | **Low–Medium.** Distinctive invented slang ("glob", "shmowzow") reads as AT. | **No canon catchphrases or invented AT slang in Track B.** Write our own vocabulary. GDD §7.3 and §9.6 achievement names need a Track B pass. |
| 8 | **Voice / VO sound-alikes** | **Recorded lines:** copyright in the sound recording and AV work (ripping clips = literal copying). **Imitating a distinctive actor's voice** for commercial use: **right of publicity / false endorsement** (*Midler v. Ford*, 9th Cir. 1988; *Waits v. Frito-Lay*, 9th Cir. 1992). **AI voice clones:** state digital-replica laws (e.g., Tennessee ELVIS Act, 2024). [Memory] | **Critical** (ripped audio or AI clone) / High (impersonation) | Low (original voices) | **No ripped audio, no AI cloning, no "do the Jake voice" casting briefs** in Track B. GDD §14.5 already bans AI cloning; extend the ban to sound-alike direction. |
| 9 | **Music** (theme song, "Island Song", other show songs, melodic sound-alikes) | Musical compositions and sound recordings: strong copyright. Melodic similarity is the classic music claim. | **Critical** | Low | Score is 100% original (GDD §14.1). **No melodic quotation.** Composer signs an originality warranty. No covers. |
| 10 | **Logos, title cards, show fonts, key art, loading screens** | Copyright (art) + trademark (logos) | **Critical** | Low–Medium (watch for title-card pastiche) | No ripped fonts or logos (GDD §12.7). Track B title cards use our own lettering. **Don't trace** art-book images (GDD §11 references Ghostshrimp backgrounds). |
| 11 | **Locations / map recreations** (Tree Fort, Candy Kingdom castle, Ice Kingdom, Marceline's cave, Time Room, Lich's Well) | Background designs are pictorial/graphic expression, and 3-D recreations are derivative works. **Generic functions** (treehouse base, candy castle, ice fortress) are ideas. | High | **Low–Medium** | Keep the **function**, change the **landmark**: no boat-in-tree lookout, no Tree Fort room layout, no Candy Castle silhouette. GDD §5.5 already says this; enforce it in level review. |
| 12 | **Lore text, episode references, Collection Book, quizzes** | **Fictional "facts" are expression** (*Castle Rock*). Plot retellings are derivative. | High | Low (original lore) | Track B lore is written from scratch. **No episode-index mapping** in shipped builds (keep GDD Appendix D internal to Track A). |
| 13 | **Mechanics** (M1 chain, block, Awakening, Realms/domains, gacha + pity, Breakaway) | **Free** (§102(b); *Data East v. Epyx*; *Capcom v. Data East*; *DaVinci*) | Low | **Low** | Copy freely from the genre. Still avoid copying JJS's *audiovisual* expression (its UI art, specific VFX) under the *Tetris* look-and-feel logic. |
| 14 | **Signature moves that restage specific episode scenes** | Thin expression per move. Strong **cumulatively** and when paired with likeness. | High | Low–Medium | In Track B, moves are **archetype-driven** ("stretch-slam"), not **episode-driven**. Drop "cite the episode" (GDD §3.3 P2) from the Track B brief. |
| 15 | **Bond Moves / duo pairings** (Finn+Jake, Marceline+PB) | Relationships and story roles add to delineation | High | Low | Original pairings in Track B. |
| 16 | **Thumbnails, icons, trailers, store copy, SEO keywords** | Trademark (confusion, implied endorsement). Copyright if they show the likeness. Roblox flags misleading metadata [Docs]. | **Critical** (they are also what rights-holder staff see first) | Low | No AT characters, palettes or "AT-like" bait in Track B marketing. Thumbnails show our own cast. No brand tags. |
| 17 | **Monetization** (Wishes for styles, passes, Club, private servers, rewarded ads) | Not IP itself, but it **turns infringement into commercial infringement**: kills fair use, raises damages and profits exposure, makes enforcement likelier. Paid random rolls for a kids' brand add **brand-safety** motivation. | **Critical multiplier** | Low | Monetize only original IP. Under Track A, the licensor sets the revenue share and rules. |
| 18 | **UGC / Marketplace items** (avatar accessories of characters) | Copyright + trademark. Separately licensed on Roblox ("Marketplace license") [Docs] | **Critical** | Low | No IP-bearing UGC unless licensed. |
| 19 | **Creator Store / Toolbox assets** | Third-party uploads may be ripped IP; a free model can still infringe. Creator Store assets must follow IP rules [Docs: `production/creator-store.md`]. | High | Medium (accidental import) | **Asset-provenance log.** No Toolbox character, map or audio assets without a provenance check. |
| 20 | **Internal docs / GDD naming** | Discoverable evidence of access, copying and willfulness | High (if Track C ships) | **Medium** (if Track B is visibly "AT with names swapped") | Keep Track A docs separate. Write the Track B bible as an **independent creative brief** from archetypes. Don't build Track B by find-and-replace on AT designs. |

### 9.1 What makes a takedown *more* or *less* likely

| More likely ↑ | Less likely ↓ |
|---|---|
| The rights holder (or its licensee) has or launches a **Roblox product in the same space** (CN has first-party Roblox games; the anime-game precedent was Gamefam/MHA) | No competing product; the IP is dormant |
| The rights holder **registers in License Manager**: daily matches, bulk removals [Docs] | The rights holder is absent from Roblox tooling |
| **Brand words in the title, tags, thumbnail or description** | Ordinary-word title, no brand terms |
| **Ripped or official assets** (trivially provable) | Original assets throughout |
| **Character likeness models** and recognizable silhouettes | Movesets on the player's own avatar; original characters |
| **Large scale / front page / press coverage / sponsored ads** | Small, quiet |
| **Monetization tied to the IP** (selling characters, gacha) | Little or no monetization; original items only |
| **Kids' brand + brand-safety concerns** (paid random items, violence, off-model behavior) | Content clearly within brand norms |
| **Active franchise moment** (new series, film, relaunch). AT has *Side Quests* S2 (Oct 2026) and a film in development (GDD §5.1). | Franchise between cycles |
| **Ownership change** (new legal teams often audit portfolios) [Inference] | Stable ownership |
| **A US/Western rights-holder culture** (WB, Paramount, Nintendo of America) | A Japanese owner with a history of tolerance |
| The dev **publicly claims** to be "the AT game" | Clear non-affiliation; no brand claims |

---

## 10. The "recognizability test" (practical tool for Track B)

**The legal question is roughly:** would an ordinary fan recognize a *specific protected character or place* from Adventure Time? [Inference, distilled from §1.3]

**Turn it into a review gate.** Before any Track B character, map or move is approved:

1. **Silhouette test.** Show a black silhouette to 5 people who know AT. If **2 or more name an AT character**, redesign.
2. **Palette + prop test.** Does it combine a canon palette with a canon signature prop (white hat + blue shirt + sword, pink + crown, blue skin + crown + beard)? If yes, redesign.
3. **Pairing test.** Does it recreate a canon *relationship* (boy hero + yellow stretchy dog, vampire rocker + candy monarch)? If yes, change the pairing.
4. **Line test.** Do any lines, names or slang match AT canon ("glob", "mathematical", "math", "shmowzow", "lumping")? If yes, cut them.
5. **Place test.** Could a fan name the AT location from a screenshot? If yes, change the landmark.
6. **Episode test.** Is any move or quest a restaging of a specific episode beat? If yes, make it archetype-driven.

**Record the result in the asset's metadata** (see §11). Failures go back to art.

---

## 11. Build rules for the Studio agent session (for the design document)

These are written as instructions an agentic Studio session can follow. [Inference, derived from §1–§10]

1. **Two content packs, one code path.** All IP-bearing data (names, epithets, models, VO, barks, lore, map dressing, UI strings) lives in a **`SkinPack`** data module. **The default pack is `Original` (Track B).** The `Licensed_AT` pack must not exist in any **public** place until a license is signed. GDD §5.3 already requires IP separation; this makes the default explicit.
2. **Banned-string lint for public builds.** A pre-publish check fails if any of these appear in place names, descriptions, `StringValue`s, `TextLabel`s, localization tables, badge or pass names, or asset names:
   - Adventure Time, Cartoon Network, Ooo, Finn, Jake, BMO, Marceline, Bubblegum, Ice King, Simon, Lich, Prismo, GOLB, Lemongrab, Peppermint Butler, Gunter, Tree Fort, Candy Kingdom, Nightosphere;
   - the canon catchphrases;
   - any other names counsel adds.

   (Ship this as a test the agent runs.)
3. **Asset provenance.** Every mesh, texture, sound and animation records `{author, source, license, recognizabilityReviewed: bool}`. **No Toolbox/Creator Store character, map or audio assets** without a provenance entry. No assets traced from show frames or art books.
4. **No ripped or derived audio. No AI voice cloning. No sound-alike casting.** Original score only, with no melodic quotation of show songs.
5. **Metadata and marketing.** Titles, descriptions, tags, thumbnails, icons, video ads and update names use only original IP. No brand keywords for SEO.
6. **Recognizability gate.** Use the §10 tests as an acceptance criterion for each style, Variant, map landmark and lore card.
7. **Docs hygiene.** Keep the Track A bible (the canon-name GDD) **separate** from the Track B production bible. Track B briefs describe **archetypes and original designs**, not "X but different". This is about creating Track B *independently* (good practice and good evidence), **not** about hiding anything. Keep all documents; never destroy records.
8. **Monetization gate.** No Robux product, Wish or UGC item may reference IP outside the active `SkinPack`. The `Licensed_AT` pack's monetization is configured only under the signed license terms.
9. **Mechanics are free; expression isn't.** Borrow genre mechanics freely (JJS/TSB grammar). Don't copy another game's UI art, VFX sprites or animations verbatim (*Tetris v. Xio* look-and-feel logic).
10. **License Manager watch.** If Cartoon Network/WBD lists *Adventure Time* in the Licenses catalog, the producer gets alerted. **Never "pre-use" AT IP in a public place while a request is pending:** "Early use of IP in your game may lead to early activation." [Docs: `ip-licensing/creators.md`]

---

## 12. Corrections and caveats for the orchestrator / GDD

- ***DaVinci v. Ziko* is a US decision (S.D. Tex. 2016), not Australian.** I found no Australian case of that name. [Web]
- ***Spry Fox v. 6waves* produced no merits ruling.** The motion to dismiss was denied, then the case settled with an IP transfer. [Web]
- ***Capcom v. Data East*:** I confirmed denial of the preliminary injunction and an August 1994 summary-judgment order exist. Secondary sources conflict on the final disposition date. [Web]
- **GDD §5.1 says JJS "still uses 'Jujutsu' in its title."** Legally, "Jujutsu" alone is a common noun; the mark is "Jujutsu Kaisen". That *lowers* the title's trademark risk. It isn't a sign of recklessness. [Inference]
- **"Algebraic!" is itself an AT catchphrase.** Its trademark status is unknown. **Add it to GDD Appendix F**, Open Questions. [Inference]

---

## 13. Open questions for counsel

1. **Current USPTO status** (live/dead, classes) of ADVENTURE TIME (serials 85477828, 77619257 and others) and any registrations for character names, LAND OF OOO, and catchphrases including **ALGEBRAIC** and **MATHEMATICAL**.
2. **Is "Algebraic!" clear to use as a game title** for an original-IP game, given its association with the show?
3. **Who controls AT licensing after the reported WBD → Paramount/Skydance merger?** Is AT (or will it be) in Roblox License Manager?
4. **Track B review:** have counsel review the final Track B character sheets and map landmarks against the recognizability test before Beta.
5. **Voice:** confirm the casting-direction language avoids right-of-publicity claims, and check digital-replica statutes in relevant states (TN, CA, others enacted in 2024–26).
6. **Japan and EU exposure** for a global Roblox launch: local law and the DSA notice path. A US-centric analysis may understate the EU's copyright/trademark regimes. [Inference]
7. **Verify the [Memory] case citations** in §1–§5 before they appear in any external document.

---

## Sources

**Roblox official docs (offline snapshot 2026-10-10, read directly)**
- `production/publishing/ip-guidelines.md`: IP policy, copyright basics, DMCA notice and counter-notice, 14-business-day restoration, trademark notices, myths list
- `production/publishing/rights-manager.md`: Rights Manager, 250-link removal requests
- `production/publishing/dsa-ip-reporting.md`: EU DSA notice and appeal
- `ip-licensing/license-manager.md`: IP families, daily matches, offers, 0–95% revenue share, 10-image minimum
- `ip-licensing/creators.md`: 14-day auto-activating offers, disputes, conditional offers, early-use warning
- `ip-licensing/faq.md`: "you must have explicit permission"; dispute → DMCA path
- `ip-licensing/get-started.md`: primary and secondary keywords and media used for scanning
- `discovery-faq.md`: misleading metadata lowers impressions
- `production/creator-store.md`: Creator Store assets must follow the IP guidelines

**Web (search snippets)**
- *DC Comics v. Towle*: https://en.wikipedia.org/wiki/DC_Comics_v._Mark_Towle ; https://www.loeb.com/en/insights/publications/2015/10/dc-comics-v-towle-dba-garage-gotham ; https://mondaq.com/unitedstates/copyright/457054/
- *Capcom v. Data East*: https://en.wikipedia.org/wiki/Capcom_U.S.A._Inc._v._Data_East_Corp. ; https://patentarcade.com/2005/08/case-capcom-v-data-east-nd-cal-1994-c.html
- *Tetris v. Xio*: https://www.loeb.com/en/insights/publications/2012/06/tetris-holding-llc-v-xio-interactive-inc ; https://newmedialaw.proskauer.com/2012/06/19/videogame-app-developer-breaks-the-rules-on-copyright-infringement/
- *Spry Fox v. 6waves*: https://www.pocketgamer.com/news/6waves-settles-with-spry-fox-in-yeti-town-triple-town-clone-debate/ ; https://mcvuk.com/development-news/triple-town-cloning-suit-settled/
- *DaVinci v. ZiKo*: https://copyrightalliance.org/copyrightability-of-board-games/ ; https://techraptor.net/content/game-mechanics-arent-protected-copyright-according-texas-court-ruling
- *Warner Bros. v. X One X*: https://www.loeb.com/en/insights/publications/2011/07/warner-bros-entertainment-et-al-v-x-one-x-produc__ ; https://www.jdsupra.com/legalnews/surrender-dorothy-court-upholds-damages-18612/
- *Disney v. Characters for Hire*: https://www.loeb.com/en/insights/publications/2018/08/disney-enterprises-inc-v-characters-for-hire-llc ; https://archive.techdirt.com/articles/20180814/09370240427/disney-gets-early-loss-trademark-copyright-suit-against-unlicensed-birthday-party-characters.shtml ; https://www.worldtrademarkreview.com/article/the-empire-strikes-out-disney-marvel-and-lucasfilm-lose-summary-judgment-trademark-infringement-claims
- AT trademarks: https://trademark.justia.com/owners/the-cartoon-network-inc-2453528/page6 ; https://trademark.justia.com/owners/the-cartoon-network-inc-796228/page11 ; https://www.trademarkelite.com/trademark/trademark-detail/87097039/APOLLO-GAUNTLET
- JJK committee warning (Dec 2025): https://respawn.outlookindia.com/pop-culture/pop-culture-news/jujutsu-kaisen-team-moves-against-unlicensed-mobile-games ; https://automaton-media.com/en/news/jujutsu-kaisen-production-team-cracks-down-on-rip-off-mobile-games-featuring-characters-oddly-similar-to-those-in-the-anime-we-will-be-taking-strict-measures/
- Shueisha anti-piracy: https://torrentfreak.com/?p=234915
- Roblox anime takedowns (2023, alleged): https://www.dualshockers.com/roblox-anime-adventures-removed/ ; https://www.pockettactics.com/anime-adventures/private ; https://www.sportskeeda.com/roblox-news/news-roblox-enforces-dmca-anime-games-face-danger ; https://x.com/Roblox_RTC/status/1738005940128100735
- *Anime Battle Arena* longevity: https://abawiki.wiki/resources/roblox-copyright/
- Blox Fruits: https://rowatcher.com/news/the-blox-fruits-timeline-from-2019-launch-to-60-billion-visits-in-2026 ; https://progameguides.com/roblox/what-happened-to-blox-fruits-recreation/
- Nintendo fan-game takedowns: https://gamesbeat.com/nintendo-allegedly-files-dmca-takedown-notice-over-more-than-550-fans-games/ ; https://www.nintendolife.com/news/2021/01/nintendo_issues_mass_dmca_takedown_379_fan-made_games_forcibly_removed
- Doujinshi / TPP: https://bilaterals.org/japanese-fan-comics-could-die ; https://www.itmedia.co.jp/news/spv/1901/17/news105.html
- Japanese guidelines: https://eventhubs.com/news/2025/nov/07/capcom-fan-content-guidelines ; https://soranews24.com/2021/02/04/nier-automatas-creator-is-confused-by-square-enixs-new-fan-art-guidelines/amp/ ; https://automaton-media.com/en/news/20230217-17921/ ; https://automaton-media.com/en/news/why-does-umamusume-pretty-derby-have-specific-rules-against-sexual-or-violent-fanart/ ; https://automaton-media.com/en/news/send-us-your-doujinshi-nitroplus-affirms-positive-stance-on-fan-made-content-asks-people-to-show-them-what-theyre-cooking/ ; https://animenewsnetwork.com/interest/2016-01-09/broccoli-lays-down-the-law-for-dojin/.97335
- Star Trek / Axanar: https://www.space.com/35436-star-trek-axanar-fan-film-lawsuit-settled.html ; https://trekmovie.com/2017/01/20/axanar-studios-settle-star-trek-copyright-infringement-lawsuit/
- WB Harry Potter festivals: https://slashfilm.com/harry-potter-fan-festivals-crackdown-warner-bros

**[Memory] (not re-verified; check before relying on any of them)**
- **Statutes:** 17 U.S.C. §§101, 102(b), 103(a), 106(2), 107, 412, 504, 505, 512(c)/(g)/(i); 37 C.F.R. §202.1(a); Lanham Act §§32, 33(b)(4), 43(a), 43(c).
- **Copyright cases:** *Nichols*; *Anderson v. Stallone*; *MGM v. American Honda*; *Daniels v. Disney*; *Castle Rock*; *Micro Star v. FormGen*; *Salinger v. Colting*; *Data East v. Epyx*; *Atari v. Philips*; *Campbell*; *Warhol v. Goldsmith*.
- **Trademark cases:** *Jack Daniel's v. VIP*; *Rogers v. Grimaldi*; *New Kids on the Block*.
- **Voice / publicity:** *Midler v. Ford*; *Waits v. Frito-Lay*; Tennessee ELVIS Act.
- **Japan:** Copyright Act Art. 123 and the 2018 conditions; the Doujin Mark (2013).
