# Enforcement risk for an Adventure Time battlegrounds game: Cartoon Network / WBD / Skydance, and how Western studios behave on Roblox

*Research note for* Algebraic! *GDD §5 (IP strategy). Date: 2026-10-12. Not legal advice.*

**Tags:** [Docs] = official Roblox creator-docs snapshot (2026-10-10), read directly · [Web] = web source (URL given) · [Memory] = prior knowledge, not re-verified this session · [Inference] = my analysis.

**Method and limits.**
- I ran 27 web searches. WebFetch failed with DNS errors for every site I tried (itch.io, videogameschronicle.com, cordcuttersnews.com, theatdb.com). Web facts below therefore come from **search-result snippets**, not full articles.
- roblox.com and the live License Manager catalog were not reachable, so I could not check the live Roblox catalog.
- Sibling notes `ip-law.md` (the law) and `roblox-cases.md` (anime and other Roblox case studies) are in the same folder. This note does not repeat them. It focuses on **who owns Adventure Time now, what that owner does, and how likely enforcement is**.

---

## 0. Bottom line

1. **The owner changed, and that matters most.** The Paramount–Warner Bros. Discovery merger is **verified**: it closed on **Tuesday 6 Oct 2026**, and the combined company trades as **Skydance** (NYSE: SKYD) with David Ellison as CEO [Web: multiple, §1]. Two integration steps put *Adventure Time* on Roblox next to Paramount's operation:
   - **WB Games and Paramount Games Studio have been merged under Tony Driscoll ("president of games and experiences").**
   - **Cartoon Network and Nickelodeon are to be combined operationally.**

   That operation has the **most documented Roblox fan-game enforcement of any Western studio**: *The Spongy Construction Project* was struck three times, and Paramount reportedly acted against "numerous" Nickelodeon-referencing games, parodies included. It also runs **the highest-earning IP-based game on Roblox** (*SpongeBob Tower Defense*, made with Wonder Works) [Web, Inference].
2. **Cartoon Network's own record on AT fan games is quiet.** Across eight targeted searches I found **no documented C&D or DMCA against any *Adventure Time* fan game**, on Roblox or elsewhere. Small AT fan games exist on itch.io. That absence is weak evidence:
   - Roblox DMCA removals of small games are rarely reported.
   - Cartoon Network has run first-party Roblox games since 2020.
   - It has licensed AT into **Fortnite twice** (Apr 2025 and Jan 2026), plus MultiVersus and Brawlhalla.

   The brand is actively licensed into game platforms, so there is a motive to protect it. [Web, Inference]
3. **Warner Bros.' pattern on fan works:** it tolerates non-commercial fan expression and **acts when fans make money, crowdfund, or overlap a commercial product.** Examples: the Harry Potter MMO crowdfund, *Dark Knight Legacy*, the Middle-earth Skyrim mod, and Mortal Kombat 1 mods. A Robux-monetized gacha game built on AT likenesses is squarely "commercial." [Web, Memory]
4. **Roblox has made enforcement cheaper.** [Docs]
   - **Rights Manager:** any verified rights holder can request removal of up to **250 links per request**, even without a Roblox presence.
   - **License Manager matching:** if Skydance registers AT, Roblox scans public, actively played games **daily**, by **keywords and images**. Skydance can then send a **license offer at a revenue share it sets, from 0–95%**. A standard offer **auto-activates in 14 days**. If the creator disputes twice, the docs name DMCA as the next step.
5. **Likelihood of a takedown, C&D or forced offer within 12 months of launch** [Inference; full table in §8]:

   | Version | Small game | Breakout hit | Hit after a trigger event |
   |---|---|---|---|
   | (a) Codename AT game, original models, no official assets | **~3–10%** | **~20–40%** | **~50–75%** |
   | (b) Same, with "Adventure Time" in the title, thumbnail or keywords | **~10–20%** | **~40–60%** | **~70–90%** |
   | (c) Ripped official art or audio | **~20–40%** | **~60–85%** | **~85–95%+** |

   The biggest trigger is **Skydance launching, or licensing a partner to launch, an official AT Roblox game**. The film and *Side Quests* make that plausible in 2027.
6. **This confirms GDD §5.3** (license-ready and IP-separable, with Track B as the default build, and **no public monetized Track C**) and sharpens it:
   - The direct-approach target is now **Skydance Games & Experiences (Driscoll) and Skydance consumer products**, not "Cartoon Network Games / WBD Global Consumer Products."
   - The most likely licensing model is **the Paramount one:** curated deals with established Roblox studios (Gamefam, Wonder Works). A newcomer needs a proven hit, and that argues for **shipping Track B first and pitching with real DAU** (§9).

---

## 1. Who owns *Adventure Time* in October 2026 (merger verification)

| Claim | Evidence | Tag / confidence |
|---|---|---|
| Paramount agreed to acquire WBD in **Feb 2026** after outbidding Netflix, at **$31.00/share cash**: about **$81B equity**, about **$110B including debt** | Jamaica Observer, 27 Feb 2026: https://www.jamaicaobserver.com/2026/02/27/paramount-acquires-warner-bros-110-bn-mega-merger/ ; esports.gg: https://esports.gg/news/gaming/paramount-warner-bros-netflix/ | [Web] High |
| The deal **closed Tuesday 6 Oct 2026**. The company is now **"Skydance"**, with Class B shares on the NYSE as **SKYD**. WBD stock stopped trading on Nasdaq. **David Ellison** is chairman and CEO. | LBC: https://www.lbc.co.uk/article/94ae9e335a3a40fc88216e2107f213c5-5HjdjTG_2/ ; Irish News: https://www.irishnews.com/news/uk/paramount-and-warner-bros-wrap-up-mega-deal-to-create-skydance-KSA57R5Q5NMYVERPDOOUKWRQYI/ ; Outlook India: https://www.outlookindia.com/international/hollywood-just-got-a-new-giant-paramount-completes-110-billion-warner-bros-deal-forms-skydance ; THR India: https://www.hollywoodreporterindia.com/features/insight/paramount-acquires-warner-bros-discovery-to-form-global-entertainment-giant-skydance ; GDD cites Axios 6 Oct 2026 | [Web] High. Several outlets agree, and 6 Oct 2026 was a Tuesday. |
| One snippet names **Ynon Kreiz** as incoming co-CEO | Single aggregator snippet | [Web] **Low.** This is surprising (Kreiz has been Mattel's CEO [Memory]). Verify before repeating. |
| **Video games merged:** WB Games' internal studios (Avalanche, NetherRealm, Rocksteady, TT Games, WB Montreal, WB Boston) and Paramount Games Studio now sit under **Tony Driscoll, "president of games and experiences."** The structure below him is not yet public. | VGC: https://www.videogameschronicle.com/news/paramount-and-warner-bros-combine-video-game-divisions-after-110bn-skydance-merger/ ; Inven Global: https://www.invenglobal.com/articles/26908/skydance-merges-paramount-and-warner-bros-games-divisions ; Shacknews: https://www.shacknews.com/article/150930/wb-games-paramount-games-studio-merger ; Insider Gaming: https://insider-gaming.com/paramount-wbds-game-divisions-to-merge-under-skydance/ | [Web] High |
| Before the merger, Paramount's games slate included *TMNT: The Last Ronin*, ***Avatar Legends: The Fighting Game***, SpongeBob and Star Trek | Search snippet (VGC / animationmagazine) | [Web] Medium. **The new owner already sells a fighting game built on its own animated IP.** |
| **Cartoon Network and Nickelodeon are to be folded together**: shared creative teams, combined ad sales, one distribution approach. Combined ratings are reportedly down more than 85% since 2016. | Cord Cutters News: https://cordcuttersnews.com/cartoon-network-nickelodeon-are-merging-raising-questions-about-their-future/ ; https://cordcuttersnews.com/?p=257347 | [Web] Medium. Reported as a plan; not confirmed as executed. |

**Implications** [Inference]
- AT's interactive and Roblox decisions will very likely sit with the same "games and experiences" leadership that runs Paramount's Roblox program: Gamefam for SpongeBob and TMNT, and Wonder Works for *SpongeBob Tower Defense*.
- Expect **Paramount's Roblox playbook** to become AT's playbook: licensed partner products plus takedowns of unlicensed look-alikes.
- **Short-term:** integration churn (Oct 2026 – mid 2027) may slow *licensing* decisions (GDD risk R2). It probably will **not** slow *takedowns*. Large studios usually run brand protection through standing legal teams and outside vendors, which keep operating through reorganizations. [Memory/Inference]

---

## 2. *Adventure Time* fan games and Cartoon Network's enforcement record

### 2.1 What exists
- **itch.io:** two tag pages (`adventuretime` and `adventure-time`) list many small AT fan games: a lollipop-collecting game, a Jake sandwich platformer, a BMO game, Lemongrab games, a dungeon crawler and a Wander Over Yonder crossover. https://itch.io/games/tag-adventuretime · https://itch.io/games/tag-adventure-time [Web]
  - Scale is small, mostly browser jam-style projects. I found no download counts.
- **Game Jolt and Newgrounds:** search returned nothing AT-specific. [Web: absence]
- **Roblox:** web search does not index Roblox experiences, and roblox.com was unreachable.
  - [Memory, low confidence] Small AT-themed morph, RP and obby games, plus UGC hats in the style of Finn's hood, have existed on Roblox. I know of **none at notable scale** and could not verify any by name.
  - The GDD's "no official AT game on Roblox" still stands. [Web: absence]
- **Official UGC precedent:** Cartoon Network once offered an official AT game-maker for fans, often recalled as the "Adventure Time Game Wizard," around 2015 [Memory]. A Gamezebo result titled "cartoon-network-launches-adventure-time-game-creator" (https://www.gamezebo.com/news/cartoon-network-launches-adventure-time-game-creator/) appeared, but its snippet did not match its title. **Low confidence.**

### 2.2 Enforcement found
| Search | Result | Tag |
|---|---|---|
| "Adventure Time Roblox game removed", "Adventure Time fan game cease and desist", "Cartoon Network DMCA" + SU / Regular Show / Ben 10, "itch.io takedowns Cartoon Network / Warner Bros / Adventure Time", "Pibby Roblox Cartoon Network copyright" | **No documented C&D or DMCA against an AT, Steven Universe, Regular Show or Ben 10 fan game.** | [Web: absence] |
| *Ben 10 Power Trip* (itch.io) | The developers **deleted it themselves for fear of a copyright claim** from the owners, and said they **had not received a strike**: https://outrightltd.itch.io/ben10powertrip/devlog/805460/the-game-is-getting-deleted (snippet only) | [Web] Medium |
| Cartoon Network's **own** licensed games | WBD **delisted** four Cartoon Network Games titles without notice. They include two AT titles (*Finn and Jake's Epic Quest*, *Magic Man's Head Games*) and two Steven Universe titles (Grumpyface). Date not established. PlayStation LifeStyle: https://www.playstationlifestyle.net/?p=912087 ; Toy People: https://www.toy-people.com/en/?p=95823. *Finn & Jake Investigations* was delisted on **1 Apr 2018**, reportedly because its license expired (MobyGames / Steam forum). In **Mar 2024** WBD delisted Adult Swim indie games, then reversed course after backlash: https://videogameschronicle.com/news/warner-bros-u-turns-on-decision-to-delist-indie-games-published-by-adult-swim ; https://gigazine.net/gsc_news/en/20240309-warner-bros-delist-games-steam | [Web] Medium |
| Cartoon Network as a litigant | It is mostly a **defendant** in reported cases (for example, a dismissed claim that *Steven Universe* copied a comic). It also has older trademark/copyright actions such as *Time Warner v. Kenafric*. https://cipit.org/time-warner-v-kenafric-copyright-and-trademark-fight-over-cartoon | [Web] Low relevance |

**Read-through** [Inference]
- Cartoon Network has historically been **tolerant or inattentive toward small, free AT fan games.** Fear of enforcement (Power Trip) exists even without enforcement. The delistings show a **rights portfolio that is actively culled and managed**, which is not the same as fan policing.
- That record describes **old WBD**. The relevant forecaster is now **Paramount's behaviour on Roblox** (§5), because Skydance's merged games and kids units will own the decision.

---

## 3. Cartoon Network / WBD activity on Roblox and on other game platforms

### 3.1 Roblox (first-party)
| Product | Facts | Tag |
|---|---|---|
| ***Ben 10: Super Hero Time*** | Launched in 2020. Reported at **260M+ / 285M+ visits** in 2023-era coverage. The GDD's "~557M" is presumably a newer figure; re-check it live. | [Web] https://kidscreen.com/?p=210333 , https://animationmagazine.net/cartoon-network-game-on-brings-gumball-haunted-house-to-roblox |
| ***Cartoon Network Game On!*** | A multi-island hub: **Craig of the Creek, Teen Titans Go!, We Baby Bears, Ben 10, Gumball.** **No AT island.** Launch is **conflicting**: Kidscreen implies **Apr 2022** (34M visits); other outlets say early 2023. | [Web] https://busrep.co.za/saturday-star/news/cartoon-network-game-on-is-live-on-roblox-9cb19a04-7220-4259-b69e-1903a3f48e75 , https://www.satellifacts.com/news/actualite/cartoon-network-launch-of-the-game-game-on!-on-the-roblox-platform |
| ***Gumball Haunted House*** and ***Gumball Tower of Defence*** (May 2024) | Both run inside Game On! A WBD executive called the approach a **"win-win for our audience and brand"** and said performance data informs the next experience. | [Web] https://www.licenseglobal.com/video-games/-the-amazing-world-of-gumball-comes-to-roblox , https://licensingsource.net/cartoon-network-game-on-launches-gumball-haunted-house-experience-and-tower-of-defence |
| *Elf* North Pole Workshop (WB Pictures, with Fairview Portals and Supersocial) | Shows WB film also commissions Roblox agencies | [Web] https://www.nasdaq.com/articles/warner-bros.-discovery-wbd-unveils-elf-north-pole-workshop |
| DC on Roblox (e.g., the *Wonder Woman* Themyscira experience, 2020) | — | [Memory] Medium |
| WBD / Cartoon Network / Skydance in **License Manager** | **None found.** Coverage lists Lionsgate, Netflix, SEGA and Kodansha. I could not see the live catalog. | [Web: absence] https://about.roblox.com/newsroom/2025/07/roblox-launches-new-licensing-platform-for-experiences |

### 3.2 *Adventure Time* licensed into games (licensing appetite)
| Deal | Facts | Tag |
|---|---|---|
| **Fortnite, Wave 1** | Live **Fri 11 Apr 2025** (Chapter 6 Season 2), Item Shop only. Skins: Finn, Jake, Princess Bubblegum, Marceline (1,500 V-Bucks each), plus BMO and LSP back blings, emotes and a glider. | [Web] https://www.pockettactics.com/fortnite/adventure-time , https://www.esports.net/news/fortnite/adventure-time-x-fortnite-cosmetics/ |
| **Fortnite, Wave 2** | **Thu 15 Jan [2026]**: Fionna, Cake, Earl of Lemongrab, with the Ice King in some form. This comes from pre-release coverage; the final lineup is unverified. | [Web] https://esports.gg/news/fortnite/adventure-time-fortnite-wave-2/ |
| **MultiVersus** (WB Games' own platform fighter) | Finn and Jake were on the launch roster (Finn revealed 18 Nov 2021). Online services **closed 30 May 2025**, and developer Player First Games was shut down. | [Web] https://multiversus.wiki.gg/wiki/Finn , https://www.thesixthaxis.com/2025/01/31/multiversus-will-go-offline-only-after-season-5/ , https://gamingbolt.com/multiversus-has-been-delisted-from-all-storefronts-servers-have-been-shut-down |
| **Brawlhalla** | **June 2019** crossover: Finn, Jake and PB as skins on existing Legends | [Web] https://gematsu.com/2019/06/brawlhalla-x-adventure-time-crossover-event-announced |

**Read-through** [Inference]
- AT is a **proven fighting and platform-game license**: MultiVersus, Brawlhalla, and Fortnite twice. WBD chose **Epic's UGC-adjacent platform**, not Roblox, for AT's 2025–26 cosmetics.
- So there is a **commercial AT game-platform relationship to protect**. There is also a **gap on Roblox** that Skydance may want to fill, plausibly with a Paramount-style partner around the film or *Side Quests*.
- A popular unlicensed AT battlegrounds game would be **direct competition for that slot**. That is the "Gamefam pattern" that preceded the alleged Anime Adventures / AFS removals (see `roblox-cases.md`).

---

## 4. Warner Bros.' general posture on fan works

| Case | What happened | Commercial trigger? | Tag |
|---|---|---|---|
| Harry Potter MMO crowdfund | Pulled from a crowdfunding site after a **WB takedown notice**. The company later renamed itself. Date not given. | Yes (crowdfunding) | [Web] https://worldipreview.com/copyright/warner-bros-sees-no-magic-in-unauthorised-harry-potter-game-8016 |
| *Dark Knight Legacy* (Nightwing fan series, Machinima, 2013) | **C&D** ended its Indiegogo campaign | Yes | [Web] https://www.tubefilter.com/2013/09/26/dark-knight-legacy-cease-and-desist-warner-bros/ |
| Middle-earth Roleplaying Project (Skyrim mod, 2012) | **C&D**; development halted | Overlapped WB's LOTR games | [Web] https://gameranx.com/updates/id/9080/article/warner-bros-issues-cease-and-desist-letter-over-middle-earth-roleplaying-project-skyrim-mod/ |
| Mortal Kombat 1 mods (2024) | A YouTuber was threatened with copyright strikes over mod videos. WB also reportedly sent DMCAs to Nexus Mods. | Overlapped paid DLC | [Web] https://videogameschronicle.com/news/warner-bros-is-threatening-to-destroy-a-popular-mortal-kombat-youtube-channel-its-claimed , https://www.gfinityesports.com/article/mortal-kombat-1-youtube-channel-sued-by-warner-bros |
| MultiVersus fan revival | **No takedown found.** Only the official shutdown (30 May 2025) is reported. | — | [Web: absence] |
| *Harry Potter Lexicon* (WB v. RDR Books, 2008) | WB sued over a **commercial** print version of a fan reference site, and won | Yes | [Memory] High |
| "Potter War" (2000–01) | WB sent letters to children's HP fan sites, faced public backlash, and softened its stance toward non-commercial fan sites | No, and WB backed off | [Memory] Medium |
| *Mortal Kombat: Rebirth* (2010 fan short) | WB then backed the director's official *Mortal Kombat: Legacy* series. **A fan work turned into a sanctioned project.** | — | [Memory] Medium |

**Pattern** [Inference]
- WB **tolerates non-commercial fan expression** (fan art, fanfic, cosplay, free jam games).
- It **acts on monetization** (crowdfunding, paid products), **overlap with its own commercial products**, and **scale or visibility**.
- It **avoids hitting children's fan communities** because of PR cost, which slightly lowers the risk for tiny free kids' games.

*Algebraic!* in Track C form would be **monetized**, with **paid character rolls (gacha) on a kids' brand**. It would **overlap a licensed category** (AT fighters in MultiVersus, Brawlhalla and Fortnite) and is **designed to scale**. It hits every trigger.

---

## 5. Other Western studios on Roblox

| Studio / IP | Official Roblox activity | Fan-game enforcement on Roblox | Notes | Tag |
|---|---|---|---|---|
| **Paramount / Nickelodeon** (**now AT's sister unit**) | *Nickverse* (Jul 2022; shut down Oct 2023 per `roblox-cases.md`). **Gamefam × Paramount:** *SpongeBob Simulator* (26 Jan 2023), *TMNT Battle Tycoon* (Aug 2023, 19M players). **Wonder Works × Paramount: *SpongeBob Tower Defense*** (Dec 2024): 633M+ visits, 9.6M peak MAU, ~113k CCU, **Best of IP award at the 2025 Roblox Innovation Awards**, and **"the highest-earning game based on existing IP on Roblox"** (Variety, 24 Apr 2026; 2.0 relaunch). | ***The Spongy Construction Project*** was taken down in **Jun 2022**, within weeks of Nickverse's 6 Jul 2022 launch; Nickverse's developers denied involvement. It was struck again in 2023. The Roblox wiki says Paramount struck **"numerous" Nickelodeon-referencing games, including parodies**. A *Survivor* game (CBS) was removed in Sep 2020. An itch.io takedown page exists for a fan game called *Spongebob: Night Shift*; the complainant could not be checked. | **The most aggressive Western enforcer on Roblox, and also the most successful licensor there.** Its model is curated partner studios. I found no License Manager listing. | [Web] https://progameguides.com/roblox/gamefam-to-partner-with-paramount-for-new-spongebob-and-teenage-mutant-ninja-turtle-experiences-but-players-arent-happy-roblox/ , https://spongyconstructionproject.fandom.com/wiki/SpongeBob_SquarePants:_The_Spongy_Construction_Project_(Roblox) , https://roblox.fandom.com/wiki/Copyright_and_trademark_infringement , https://games.gg/news/spongebob-tower-defense-on-roblox-hits-113k-ccu/ , https://news.designrush.com/paramount-spongebob-roblox-tower-defense-2-launch-wonder-works , https://playday.one/2026/04/24/spongebob-tower-defense-2-0-now-live-on-roblox/ , https://www.mediapost.com/publications/article/382535 , https://itch.io/takedowns/3287197 |
| **Disney** (Marvel, Star Wars, Pixar) | Limited Roblox presence. Disney's big UGC bet was **Epic: a $1.5B investment in Feb 2024** [Memory]. | **No Roblox case found.** Off Roblox: a **DMCA shut down *Club Penguin Online*** (fan server, about 8M users, 2020) after an arrest tied to the site; Disney cited child safety. | Child-safety framing is a powerful trigger for kids' brands | [Web] https://techraptor.net/gaming/news/club-penguin-fan-server-club-penguin-online-shut-down-by-disney , https://archive.techdirt.com/articles/20200515/11250044510/disney-if-we-cant-run-club-penguin-no-one-can-run-club-penguin-updated.shtml |
| **Hasbro** (MLP, Transformers) | None found | No Roblox case found. **Key analog:** the fan **fighting game *My Little Pony: Fighting is Magic*** received a **C&D in Feb 2013**. It relaunched as original IP, ***Them's Fightin' Herds***, with Lauren Faust designing the cast, and raised $500k+ from 10,513 Indiegogo backers in 2015. | **The closest precedent for our exact case:** a fan fighter on a kids' cartoon brand, stopped, then reborn as Track B and commercially viable. | [Web] https://www.gameinformer.com/b/news/archive/2013/02/09/hasbro-shuts-down-unlicensed-my-little-pony-fighting-game.aspx , https://www.techdirt.com/2015/09/25/hasbro-spent-time-money-lawyers-attention-to-barely-make-difference-over-my-little-pony-fan-game |
| **SEGA** (Sonic) | *Sonic Speed Simulator* (Gamefam, Apr 2022). License Manager launch partner. | Took down *Sonic Eclipse Online* (see `roblox-cases.md`) | Fan-tolerant overall, but enforced after a licensed product launched | [Docs, Web] |
| **Mattel** | "Polly Pocket" is the docs' worked example of an IP family and keywords | — | Suggests Mattel works with License Manager | [Docs, Inference] Low–Med |
| **Netflix, Lionsgate** | License Manager catalog partners (Stranger Things, Squid Game, Twilight, horror IPs) | — | License-first model | [Docs] `ip-licensing/index.md`; [Web] |
| **Nintendo** (non-Western, but the reference case) | None | *Pokémon Brick Bronze* (2018); several Mario games (Aug 2018); Roblox later filtered those names | — | [Web] via `roblox-cases.md` |

**Pattern across Western studios** [Inference]
1. Studios with **their own Roblox revenue** in a genre (Paramount, SEGA, Nickelodeon) are the ones that **remove look-alikes**, often around a launch.
2. **Kids' brands** add **brand-safety triggers** such as gacha, violence and child safety. A battlegrounds game where children's cartoon characters beat each other up, sold through randomized paid rolls, is the kind of thing a kids' licensor does not want circulating unofficially.
3. **License-first platforms are emerging** (Netflix, Lionsgate, SEGA and Kodansha on License Manager), but **Paramount has not been seen using License Manager**. Its model is direct deals with top Roblox studios.

---

## 6. Roblox mechanics that set the enforcement path [Docs]

Read directly from `content/en-us/ip-licensing/*.md`, `production/publishing/rights-manager.md` and `production/publishing/ip-guidelines.md`.

**Rights Manager (removal)**
- Open to any registered, verified rights holder **with a registered copyright or trademark**, including those without a Roblox presence.
- One removal request covers **up to 250 links** to games, avatar items or assets. Status is Pending, Approved (removed) or Rejected.
- Removals are cheap and can be done in bulk.

**DMCA**
- A valid notice leads to a fast takedown. A counter-notice usually restores content **within 14 business days** unless the claimant sues.
- **Knowingly false counter-notices create liability under §512(f).** For a game that really does use AT characters, a counter-notice is not a realistic defence.
- Roblox **suspends or terminates accounts of repeat or egregious infringers.** This threatens the **group or account that owns our other games**.

**License Manager eligibility (rights-holder side)**
- Exclusive, worldwide rights.
- A commercialized portfolio of 5+ works, or an existing license with an established Roblox creator.
- Skydance clearly qualifies for AT. [Inference]

**Matching ("reactive licensing")**
- The rights holder creates an **IP family**: a primary keyword (e.g., "Adventure Time"), secondary keywords (characters, places) and **at least 10 approved images**.
- Roblox matches **daily** against **public, actively played games** "that use a significant amount of content related to your IP." The rights holder also sees **gameplay screenshots captured by Roblox**.
- Matching covers **perpetual licenses only**. The rights holder can also file **3 manual match requests a day**.
- *Codenames defeat keyword matching only. Image matching and Roblox's own screenshots can still flag a recognizable Finn or Marceline.* [Inference]

**Offers**
- A **standard offer auto-activates after 14 days** unless the creator disputes it. The revenue share is set by the rights holder at **0–95%** of the creator's gross Robux, after platform fees.
- A **conditional offer** requires changes within 14 days and does not auto-activate.
- The creator may dispute **twice**. After that, the docs say the rights holder "might choose to pursue an IP infringement claim," meaning DMCA.
- The platform's communication rules **prohibit negotiating terms in-app.**

**Requesting a license (creator side)**
- Only possible if the IP is **listed in the catalog**. AT is not, as far as I could find.
- The game must have **the IP central to the whole game**.
- A rights holder can gate requests by **minimum 7-day DAU (>1,000 or >25,000)** and maximum maturity.
- *"If early IP usage is detected in a creator's game, the license request or accepted license agreement may be activated early."*

**Official guidance**
- *"If you want to use an IP in your game, you must have explicit permission from the rights holder. We recommend removing any content that might use IP and isn't covered by a license."* (`faq.md`)
- *"IP licensing provides rights holders and creators with an alternative to DMCA takedowns."* (`faq.md`)

**Other**
- Licensed games get an automatic **licensed badge**, which is a marketing benefit of Track A. Revenue share applies only after activation.
- **Self-serve timing:** `roblox-cases.md` cites Nov 2025 for "open to all IP owners," while my searches surfaced coverage dated around **June 2026** of a "self-serve IP licensing" expansion (games.gg: https://games.gg/pt-BR/news/roblox-apresenta-licenciamento-de-ip-self-serve/). They may be two phases. **Reconcile before quoting.** [Web] Low–Med
- **The 10–25% "typical" revenue share** comes from secondary coverage, not the docs. [Web] Medium

---

## 7. "How do JJS, Blox Fruits and the anime games get away with it?" (Western-IP angle)

The full answer is in `ip-law.md` §7 and `roblox-cases.md` §0. In short [Inference, consistent with both]:

- **The user's model is half right.**
  - Renaming and avoiding official art **reduce trademark exposure and automated keyword detection**. They also remove the easiest copyright evidence (ripped assets, which are what got Anime Adventures, Brick Bronze and Shinobi Life 2 into trouble).
  - Renaming **does not make a recognizable character non-infringing.**
- **The bigger reason those games survive is who owns the IP.** Japanese manga publishers have historically tolerated fan works and have **no Roblox product competing in that slot**.
  - *Blox Fruits* goes further: it is mostly **"inspired-by"** (original characters plus a One Piece-like premise), which is effectively Track B.
- **AT's owner is in the opposite category.** As of 6 Oct 2026, AT shares a parent and a games boss with Nickelodeon's Roblox program, which has the most documented Roblox takedowns of any Western studio and the platform's top-grossing IP game. **The JJS playbook does not transfer to AT.**

---

## 8. Likelihood assessment

**Definition:** the probability of **at least one adverse rights-holder action within 12 months of public launch**: a DMCA or Rights Manager removal, a C&D, or a License Manager offer we cannot accept.

**Assumptions:** monetized, with paid Wishes; JJS-style gameplay; launched in 2027. **All numbers are [Inference]:** judgement from the precedents above, not statistics. Read them as a **gradient**, not as point estimates.

**Scale tiers**
- **Small:** under about 500 typical CCU, little revenue.
- **Breakout:** about 5k–50k CCU, on discovery sorts, YouTube and TikTok coverage.
- **Hit + trigger:** a breakout **and** one of the trigger events below.

| Scenario | Small | Breakout | Hit + trigger | Most likely form of action |
|---|---|---|---|---|
| **(a) JJS-style codename game.** Original models that are still **recognizable AT characters** under epithets ("The Last Human"). No AT words in the title, thumbnail or keywords. No official assets. | **3–10%** | **20–40%** | **50–75%** | A DMCA or Rights Manager removal if Skydance has a partner AT Roblox game. A license or conditional offer if AT is on License Manager. The offer could require removing gacha and could set any revenue share. |
| **(b) As (a), but "Adventure Time" (or Finn, Jake, Ooo) in the title, thumbnail, description, tags, group name or Discord** | **10–20%** | **40–60%** | **70–90%** | Trademark plus copyright. Trivially found by keyword search, brand-protection vendors and License Manager keywords. It is also the most likely to draw a **parent or press complaint**, and press articles name the brand. |
| **(c) Ripped official art or audio** (show renders, theme song, VO clips, SFX, official logo) | **20–40%** | **60–85%** | **85–95%+** | Fastest and least defensible. Asset-level removals hit the art and audio themselves, and **account strikes** pile up (repeat-infringer policy). [Memory] Roblox screens audio uploads for copyrighted content, so some rips may be blocked at upload. |
| *Reference: Track B (original "Land of Whatever" world, no AT likeness)* | <1% | ~1–3% | ~2–5% | Only if designs drift toward substantial similarity, or someone files a nuisance or competitor claim (Vox Seas-style). |

**Why the numbers sit where they do** [Inference]
- **Low floor for small games:** most small infringing games are never touched, and Cartoon Network has no documented AT fan-game enforcement.
- **Steep rise with success:** every documented Western Roblox case (Brick Bronze, Spongy, Sonic Eclipse, Kaiju Universe) hit a **visible** game. *Algebraic!* is designed to be visible.
- **Ceiling set by owner type:** Paramount's record is "struck repeatedly, even after the game returned," plus "numerous" Nickelodeon-referencing games including parodies. The new owner does not let things slide once it notices.

**What a removal would cost** [Inference]
- The game, all Robux-backed progress, and refund backlash (Brick Bronze).
- Our publishing group's standing (repeat-infringer strikes).
- Our chance of ever getting the license. Track C makes us an **infringer** in the licensor's eyes, not a **prospective partner**.

### Triggers that raise risk
Ordered by estimated impact. [Inference, grounded in the cited cases]
1. **An official or licensed AT Roblox game is announced or launched** (Skydance in-house, or with Gamefam or Wonder Works). Precedents: Nickverse → Spongy; Gamefam MHA → AA/AFS; Sonic Speed Simulator → Sonic Eclipse. **It is plausible around the AT film or *Side Quests* S2/S3 marketing.**
2. **AT registered as an IP family in Roblox License Manager or Rights Manager.** Daily matching starts, and vendors can bulk-file 250 links. This may be invisible to us until an offer or removal arrives.
3. **Revenue:** paid **randomized rolls on kids' characters**. This is commercial, and it is a brand-safety issue.
4. **Scale and visibility:** front-page sorts, CCU spikes, a top-earning chart position, or big YouTuber coverage. AT fans are a large, nostalgic 18–30 audience on TikTok and YouTube, so **AT content travels off-platform**, where licensing staff see it.
5. **Press that names the brand** ("an Adventure Time battlegrounds game is blowing up on Roblox"). Press uses the real name even if we don't. So **(a) drifts toward (b) as it succeeds.**
6. **Off-model or brand-unsafe content:** horror-tier Lich or GOLB, gore, romance between characters, or crude humour. Any parent complaint becomes a child-safety story, the Club Penguin Online trigger.
7. **Merch-like monetization:** UGC or Marketplace items with AT likeness, or an AT logo on the group or Discord. Marketplace items are matched separately and are easy to remove.
8. **Competitor or rival reports:** anyone can file, and copycats get DMCA'd (Vox Seas, the AA developer's own DMCA).
9. **Integration "clean-up" waves:** new owners often audit the portfolio, as WBD did with its own Cartoon Network and Adult Swim games in 2024–25.

### Factors that lower risk
Within Track C, these reduce risk but do not remove it. [Inference]
- Strictly no AT words in any metadata.
- No ripped assets or audio.
- Silhouettes changed beyond recognition (at which point this is Track B).
- No paid rolls.
- Keep it free and small.
- Keep internal AT placeholder builds in **unpublished or private places owned by a separate test account or group**.
  - [Docs] Matching covers only *public, actively played* games, but **Rights Manager can target hosted assets**, so do not upload AT placeholder meshes or decals to the production group.

**Note on the user's "cosmetic look optional" change** (consistent with `ip-law.md` §8):
- A **player toggle does not reduce legal exposure if AT-likeness assets ship in the game.**
- If the IP look were **off by default**, Roblox-captured match screenshots *might* not show it, but players' videos would. Detection risk falls slightly; liability does not change. [Inference]

---

## 9. Recommendations for the design document

1. **Keep GDD §5.2–5.3 as written:** Track A target, Track B default build, **no public monetized Track C.** The evidence strengthens it, because the owner is now Paramount-run Skydance.
2. **Update the licensing contact in §5.3 step 2.** Replace "Cartoon Network Games / WBD (Skydance) Global Consumer Products" with: **"Skydance Games & Experiences (Tony Driscoll), and Skydance consumer products / the combined Nickelodeon + Cartoon Network kids group. The org chart is in flux after the 6 Oct 2026 close."** Add the Paramount Roblox partner precedents (Gamefam, Wonder Works) as the **model a pitch should mirror**.
3. **Sequence: Track B first, then pitch with metrics.**
   - Skydance's Roblox licensing pattern favours **established studios with proven DAU**. License Manager eligibility gates also use DAU (>1k or >25k).
   - A Track B hit with original mechanics, Mild maturity and transparent odds is the **strongest pitch asset**, and the IP-separable skin layer is exactly what a **conditional offer** would ask for.
   - Precedents: *Them's Fightin' Herds* (MLP fighter reborn as original IP) and *Loomian Legacy* / *Shindo Life* (pivoted after enforcement).
4. **Add a "trigger watch" to risk register R2:**
   - Monitor for an AT Roblox announcement by Skydance or a partner.
   - Monitor the License Manager catalog for an AT, Cartoon Network or Skydance listing.
   - Track Skydance games-unit hires and partner deals.
   - Track AT film and *Side Quests* marketing beats.
   - **If an official AT Roblox game is announced, any AT-likeness content still in our public game must go immediately.**
5. **Metadata hygiene even on Track B:** never use "Adventure Time," "Finn," "Jake," "Ooo," "Algebraic" with AT styling, the AT logo font, or the AT palette in a thumbnail.
   - "Algebraic!" is itself a famous AT catchphrase, so **ask counsel whether the title itself is a trademark or confusion risk on Track B** (see `ip-law.md` §3).
6. **Protect the publishing group:** keep any AT placeholder art out of the production group (repeat-infringer policy). Never file a DMCA counter-notice on AT content (§512(f)).

---

## 10. Corrections and flags for the GDD

- **§5 header / §1 "reported" merger:** **verified** (closed 6 Oct 2026, "Skydance," NYSE: SKYD). The GDD can say "completed" instead of "reported." Add the **games-division merger under Driscoll** and the **planned Cartoon Network + Nickelodeon combination**.
- **§5.1 Ben 10 *Super Hero Time* ~557M visits:** my sources show 260–285M (2023 era). Plausibly grown since; **re-check live.**
- **§5.1 *Cartoon Network Game On!*:** launch date conflicts (Apr 2022 vs 2023). It **has no AT island**.
- **§5.1, add AT licensing evidence:** Fortnite Wave 1 (11 Apr 2025) and Wave 2 (15 Jan 2026), MultiVersus (closed 30 May 2025), Brawlhalla (2019).
- **§5.1 precedents, add:**
  - Paramount's *Spongy Construction Project* takedowns (2022–23) and "numerous" Nickelodeon-referencing removals.
  - *SpongeBob Tower Defense* as the highest-earning IP game on Roblox (Apr 2026).
  - *MLP: Fighting is Magic* → *Them's Fightin' Herds*.
- **§5.1 License Manager:** "typically 10–25%" is secondary-source. The docs say **0–95%, set by the rights holder**, and a standard offer **auto-activates after 14 days**. Self-serve date: Nov 2025 vs Jun 2026 coverage; reconcile.

---

## Open questions

1. **Live License Manager catalog** (needs a Creator Hub login): is there any AT, Cartoon Network, Skydance, Paramount or Nickelodeon listing? Does Paramount use License Manager at all?
2. Is an **official AT Roblox project** in development (Skydance in-house, Gamefam, Wonder Works or another partner), possibly timed to the film or *Side Quests*?
3. After the merger, **who owns Roblox licensing for Cartoon Network IP**: Driscoll's "games and experiences," consumer products, or the combined kids group? Who is the right first contact?
4. Does the **Fortnite AT deal** carry any exclusivity over UGC-platform cosmetics or experiences that would block a Roblox license?
5. **Existing AT fan games on Roblox:** names, scale and status. Needs a live Roblox search, which was unreachable.
6. The complainant behind the itch.io takedown of *Spongebob: Night Shift*, and any itch.io or Game Jolt takedowns for AT (itch.io was DNS-blocked).
7. **Ynon Kreiz as Skydance co-CEO** (single source): verify.
8. Counsel: is the title **"Algebraic!"** (an AT catchphrase) a trademark or confusion risk even on Track B?

---

## Sources (web)

**Merger and integration**
- https://www.lbc.co.uk/article/94ae9e335a3a40fc88216e2107f213c5-5HjdjTG_2/
- https://www.irishnews.com/news/uk/paramount-and-warner-bros-wrap-up-mega-deal-to-create-skydance-KSA57R5Q5NMYVERPDOOUKWRQYI/
- https://www.outlookindia.com/international/hollywood-just-got-a-new-giant-paramount-completes-110-billion-warner-bros-deal-forms-skydance
- https://www.hollywoodreporterindia.com/features/insight/paramount-acquires-warner-bros-discovery-to-form-global-entertainment-giant-skydance
- https://www.jamaicaobserver.com/2026/02/27/paramount-acquires-warner-bros-110-bn-mega-merger/
- https://www.videogameschronicle.com/news/paramount-and-warner-bros-combine-video-game-divisions-after-110bn-skydance-merger/
- https://www.invenglobal.com/articles/26908/skydance-merges-paramount-and-warner-bros-games-divisions
- https://www.shacknews.com/article/150930/wb-games-paramount-games-studio-merger
- https://insider-gaming.com/paramount-wbds-game-divisions-to-merge-under-skydance/
- https://cordcuttersnews.com/cartoon-network-nickelodeon-are-merging-raising-questions-about-their-future/

**Cartoon Network / WBD on Roblox**
- https://kidscreen.com/?p=210333
- https://animationmagazine.net/cartoon-network-game-on-brings-gumball-haunted-house-to-roblox
- https://www.licenseglobal.com/video-games/-the-amazing-world-of-gumball-comes-to-roblox
- https://licensingsource.net/cartoon-network-game-on-launches-gumball-haunted-house-experience-and-tower-of-defence
- https://busrep.co.za/saturday-star/news/cartoon-network-game-on-is-live-on-roblox-9cb19a04-7220-4259-b69e-1903a3f48e75
- https://www.nasdaq.com/articles/warner-bros.-discovery-wbd-unveils-elf-north-pole-workshop

**AT licensing**
- https://www.pockettactics.com/fortnite/adventure-time
- https://www.esports.net/news/fortnite/adventure-time-x-fortnite-cosmetics/
- https://esports.gg/news/fortnite/adventure-time-fortnite-wave-2/
- https://multiversus.wiki.gg/wiki/Finn
- https://gamingbolt.com/multiversus-has-been-delisted-from-all-storefronts-servers-have-been-shut-down
- https://gematsu.com/2019/06/brawlhalla-x-adventure-time-crossover-event-announced

**Delistings**
- https://www.playstationlifestyle.net/?p=912087
- https://www.toy-people.com/en/?p=95823
- https://videogameschronicle.com/news/warner-bros-u-turns-on-decision-to-delist-indie-games-published-by-adult-swim
- https://gigazine.net/gsc_news/en/20240309-warner-bros-delist-games-steam

**AT fan games**
- https://itch.io/games/tag-adventuretime
- https://itch.io/games/tag-adventure-time
- https://outrightltd.itch.io/ben10powertrip/devlog/805460/the-game-is-getting-deleted

**WB enforcement**
- https://worldipreview.com/copyright/warner-bros-sees-no-magic-in-unauthorised-harry-potter-game-8016
- https://www.tubefilter.com/2013/09/26/dark-knight-legacy-cease-and-desist-warner-bros/
- https://gameranx.com/updates/id/9080/article/warner-bros-issues-cease-and-desist-letter-over-middle-earth-roleplaying-project-skyrim-mod/
- https://videogameschronicle.com/news/warner-bros-is-threatening-to-destroy-a-popular-mortal-kombat-youtube-channel-its-claimed
- https://www.gfinityesports.com/article/mortal-kombat-1-youtube-channel-sued-by-warner-bros

**Paramount / Nickelodeon on Roblox**
- https://progameguides.com/roblox/gamefam-to-partner-with-paramount-for-new-spongebob-and-teenage-mutant-ninja-turtle-experiences-but-players-arent-happy-roblox/
- https://spongyconstructionproject.fandom.com/wiki/SpongeBob_SquarePants:_The_Spongy_Construction_Project_(Roblox)
- https://roblox.fandom.com/wiki/Copyright_and_trademark_infringement
- https://www.mediapost.com/publications/article/382535
- https://games.gg/news/spongebob-tower-defense-on-roblox-hits-113k-ccu/
- https://news.designrush.com/paramount-spongebob-roblox-tower-defense-2-launch-wonder-works
- https://playday.one/2026/04/24/spongebob-tower-defense-2-0-now-live-on-roblox/
- https://www.theatdb.com/news/spongebob-tower-defense-highest-earning-ip-based-game-on-roblox-launches-20-version-in-partnership-with-paramount-exclusive
- https://itch.io/takedowns/3287197

**Disney / Hasbro**
- https://techraptor.net/gaming/news/club-penguin-fan-server-club-penguin-online-shut-down-by-disney
- https://www.gameinformer.com/b/news/archive/2013/02/09/hasbro-shuts-down-unlicensed-my-little-pony-fighting-game.aspx
- https://www.techdirt.com/2015/09/25/hasbro-spent-time-money-lawyers-attention-to-barely-make-difference-over-my-little-pony-fan-game

**Anime / Gamefam pattern**
- https://www.pockettactics.com/anime-adventures/private

**License Manager news**
- https://about.roblox.com/newsroom/2025/07/roblox-launches-new-licensing-platform-for-experiences
- https://games.gg/pt-BR/news/roblox-apresenta-licenciamento-de-ip-self-serve/

**Roblox docs read** (offline snapshot, `content/en-us/`):
- `ip-licensing/index.md`, `get-started.md`, `license-manager.md`, `creators.md`, `faq.md`, `glossary.md`
- `production/publishing/rights-manager.md`, `production/publishing/ip-guidelines.md`
