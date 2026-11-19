# CheatLab FPS – Design-Dokument

*Arbeitstitel: „CheatLab“, der Name ist noch offen. Stand: 2026-11-19, Version 2. Version 1 („externe Cheats + Kernel-Treiber“) wurde vom Nutzer verworfen.*

## 1. Idee und Warum

Ein First-Person-Shooter in der Art von *Pure Skill*: Man spielt Matches, verdient Credits und wird mit Cheats immer stärker. **Der Unterschied:** Cheats werden nicht gekauft, sondern **im Spiel selbst in C++ programmiert**. Es gibt einen Code-Editor im Spiel, man drückt auf „Ausführen“, und der eigene Aimbot, ESP oder Bhop läuft sofort. Nichts läuft extern, alles passiert im Spiel.

Ein eingebautes Anti-Cheat mit 5 Stufen prüft den Code und das Verhalten des Spielers. **Wer besser programmiert, hat den besseren Cheat** und wird seltener erwischt.

Ziel: **Spaß + C++ lernen**, für komplette Anfänger.

## 2. Ziele, Nicht-Ziele, Anti-Ziele

**Ziele**
- Movement und Gefühl wie bei Counter-Strike-HvH: Bhop, Air-Strafing, hohes Tempo.
- Eine Aufstiegs-Schleife wie bei Pure Skill: Match → Credits → neue Programmier-Werkzeuge → stärkere Cheats → schwerere Gegner und Lobbys.
- Anfänger lernen C++ von null, bis hin zu Zeigern, Klassen und Algorithmen (Zielvorhersage, Smoothing).
- 5 Anti-Cheat-Stufen im Spiel.
- Zuerst Einzelspieler gegen Bots (einstellbare Schwierigkeit, Bots cheaten nie), später Online-HvH.
- Steam-Release (Preis später), Deutsch und Englisch.

**Nicht-Ziele**
- Echte externe Cheats, DLL-Injection, Kernel-Treiber: Das gibt es nicht mehr.
- Volles C++ mit Standardbibliothek. Es gibt eine Teilmenge, siehe 5.
- Linux/Mac, große Grafik, viele Waffen.
- Map-Editor zum Start. Das Map-Format wird aber offen geplant, damit eigene Maps später möglich sind.

**Anti-Ziele**
- Spieler-Code kann aus dem Spiel ausbrechen, also Dateien, Netzwerk oder den PC anfassen. Er darf nur die Spiel-API benutzen.
- Online kann ein Skript anderen Spielern schaden, außer im Spiel selbst (Treffer usw.), oder den Server abstürzen lassen bzw. lahmlegen.
- Anfänger kommen nach den ersten Missionen nicht weiter, weil die Lernkurve zu steil ist oder die Fehlermeldungen unverständlich sind.
- Das Spiel wird zu einer Anleitung gegen echte Anti-Cheats. Die Anti-Cheat-Stufen sind Spielmechaniken von CheatLab.

## 3. Rahmenbedingungen

- **Engine: s&box** (Facepunch, Source 2), vom Nutzer gekauft. Spiellogik in C#.
- **Closed Source hat Vorrang (Entscheidung des Nutzers).** Auf der normalen s&box-Plattform wird der C#-Code mit ausgeliefert, deshalb ist das Ziel **von Anfang an der Standalone-Export**: Steam mit Valve-Freigabe und Facepunch-Lizenz. Die Freigabe wird in Phase 0 angefragt. **Plan B:** ein anderer Launcher oder ein Standalone-Release über GitHub Releases (nur die Binärdateien). Native DLLs sind trotzdem nicht nötig, weil alles im Spiel in C# läuft.
- **MCP:** „Claude Bridge“ (s&box-Editor-MCP auf `http://127.0.0.1:7269/mcp`) und der Doku-MCP `sdocs.suiram.dev`.
- Für den Standalone-Export müssen die Citizen-Standardmodelle durch eigene ersetzt werden, **deshalb gleich ab Phase 1 mit eigenen bzw. gekauften Modellen arbeiten**.
- **Team:** nur der Nutzer + Claude. Modelle und Sounds werden gekauft oder sind kostenlos (Low-Poly).
- Nur Windows (bei s&box ohnehin).

## 4. Ansätze

**Engine:** s&box (Wahl des Nutzers). Unreal und Godot wurden betrachtet. Mit „alles im Spiel“ fällt der Hauptgrund gegen s&box weg (Valve-Freigabe für native DLLs).

**Wie C++ im Spiel läuft:**

| Ansatz | Vorteile | Nachteile |
|---|---|---|
| **Eigener C++-Interpreter in C#** ✅ gewählt | Läuft auf der s&box-Plattform, sicher abgeschottet, eigene anfängerfreundliche Fehlermeldungen, simulierter Speicher möglich, online deterministisch | Nur eine Teilmenge von C++, großer Bauaufwand |
| Echter Compiler (z. B. clang → WASM) | 100 % C++ | Native DLLs → Standalone + Freigabe nötig, groß, schwer abzusichern |

## 5. Die Programmier-Sprache im Spiel („CheatLab C++“)

- **Kann:** `int`, `float`, `bool`, `char`, Arrays, `std::string` (vereinfacht), `if/else`, `for/while`, Funktionen, `struct`/`class` (Methoden, Konstruktoren), **Zeiger und Referenzen**, `const`, `#include` von Spiel-Headern, `std::vector` (vereinfacht), Mathe (`sqrt`, `atan2` …).
- **Kann nicht (vorerst):** Templates selbst schreiben, Vererbung mit virtuellen Funktionen (eventuell später), Ausnahmen, echte Standard-Header, Datei- oder Netzwerkzugriff.
- **Gültiges C++:** Was der Interpreter annimmt, soll auch echter C++-Code sein. Lernen soll sich auf echtes C++ übertragen.
- **Ausführung:** Der Spieler-Code hat Einstiegs-Funktionen wie `void OnTick(Game& game)`, `void OnDraw(Draw& draw)` und `Input OnInput(Input in)`. Es gibt ein **Rechenbudget pro Tick**. Wer es überschreitet (z. B. mit einer Endlosschleife), dessen Skript wird gestoppt und bekommt eine Meldung. Das Spiel friert nicht ein.
- **Lesen, Steuern, Schreiben:** Lesen (Positionen, Leben …) und Steuern (Zielen, Springen, Schießen) gibt es von Anfang an. **Schreiben** (Speedhack, No Recoil, Munition …) wird spät freigeschaltet, ist teuer und wirkt **nur auf den eigenen Spieler**. Das Anti-Cheat prüft es stark. God Mode und Teleport sind möglich, werden aber ab Stufe 2 sofort erkannt. Werte anderer Spieler kann man nie schreiben. Der virtuelle Speicher hat deshalb von Anfang an beschreibbare Bereiche (eigener Spieler) und schreibgeschützte Bereiche (alles andere).
- **Simulierter Speicher:** Spieldaten sind auch als virtueller Speicher erreichbar (`ReadMemory<int>(address)`, `localPlayer + 0x100`). Am Anfang benutzt man die bequeme API (`game.LocalPlayer().health`). Später schaltet man den „rohen“ Speicher frei: Er ist schneller und mächtiger, aber das Anti-Cheat erkennt die bequeme API leichter (siehe 7). So lernt man Zeiger, Offsets und Structs.
- **Editor:** Syntax-Farben, Auto-Vervollständigung für Spiel-Funktionen, Fehler mit Zeile und einer Erklärung auf DE/EN, mehrere Skript-Dateien („Cheat-Module“), die man einzeln an- und ausschalten kann.

## 6. Fortschritt: Credits → Werkzeuge

- Matches bringen Credits: Kills, Siege, Unentdeckt-Bonus.
- Im **Werkzeug-Shop** kauft man **API-Funktionen und Fähigkeiten**, keine fertigen Cheats. Beispiele:
  - `GetEnemies()` (für ESP/Aimbot), `WorldToScreen()` (für ESP-Boxen), `SetViewAngles()` (für Aimbot), `Raycast()` (Sichtbarkeits-Check, Triggerbot), `IsOnGround()` und `Jump()` (Bhop), `GetVelocity()` (Zielvorhersage), `Build()` (Auto-Build im BR), `ReadMemory()` (Zugang zum rohen Speicher).
  - Upgrades: höheres Rechenbudget pro Tick, mehr Module gleichzeitig, mehr Zeichen-Funktionen.
- Missionen geben das Wissen, wie man ein Werkzeug benutzt (siehe 8).
- Schwierigkeits-Leiter wie bei Pure Skill: 1v1-Aim-Map → kleine Lobbys → volle Lobbys mit schweren Bots. **Bot-Leiter und AC-Stufe sind zwei getrennte Achsen.**
- **Der Fortschritt wird auf dem Server des Nutzers gespeichert**, damit man ihn nicht fälschen kann. Er gilt im Einzelspieler und online, der Login läuft über Steam, und zwar **schon ab Phase 1**. Dafür braucht es ein kleines Backend (Konto, Credits, freigeschaltete Werkzeuge). Ohne Internet ist der Einzelspieler spielbar, verdient aber keine Credits. *(Claudes Entscheidung, korrigierbar)*

## 7. Anti-Cheat (5 Stufen, im Spiel)

**Auswahl:** Eine Stufe wird freigeschaltet, wenn man auf der Stufe darunter ein Match unentdeckt gewinnt. Danach kann man jede freigeschaltete Stufe frei wählen. Online hat jede Lobby eine Stufe für alle. Höhere Stufen bringen mehr Credits und mehr Unentdeckt-Bonus.

**Erwischt werden** *(Claudes Entscheidung, korrigierbar)*: Das AC führt einen **Verdachtswert**. Jede Erkennung erhöht ihn, und alle sehen die Meldung („X erkannt: Snap-Aim“). Folgen nach Stärke: (1) Match-Punktabzug, (2) Unentdeckt-Bonus verloren, (3) bei hohem Verdacht wird das erkannte Modul für den Rest des Matches abgeschaltet. Man bleibt im Spiel, aber offensichtliches Cheaten zahlt sich nie aus. Hat man eine Stufe geschafft, werden die Erkennungsregeln mit Erklärung freigeschaltet.

| Stufe | Was das AC prüft | Was man lernt |
|---|---|---|
| 1 Off | Nichts | Grundlagen, erste Cheats |
| 2 Leicht | Offensichtliches Verhalten: 180°-Snaps, Treffer durch Wände ohne Sicht, perfekte Bhop-Ketten | Smoothing, Bedingungen, Timing |
| 3 Mittel | „Signaturen“ im Code: Benutzung der bequemen Cheat-APIs wird geloggt, bekannte Code-Muster werden erkannt | Roher Speicher statt API, Zeiger, eigener Code-Stil, Verschleierung |
| 4 Schwer | Statistik: Reaktionszeit, Zielverlauf, Kopfschuss-Quote, Speicher-Prüfsummen, verschlüsselte Werte im virtuellen Speicher | Menschlich wirkender Aim (Kurven, Zufall), Entschlüsseln |
| 5 Experte | Alles + simulierter „Kernel-Schutz“: Speicher-Layout ändert sich jede Runde, Zugriffe werden gezählt, Honeypot-Werte (Fallen-Adressen) | Pattern-Scanning im virtuellen Speicher, sehr sauberer Code, Algorithmen |

- Die Stufen 4 und 5 sollen **„todesschwer“** sein (Wunsch des Nutzers), aber schaffbar.
- Das AC sieht im Einzelspieler alles. Online läuft es auf dem Server (siehe 10).

## 8. Lernsystem (Missionen)

- Missionen werden der Reihe nach freigeschaltet. Jede hat eine Erklärung, ein Ziel, Tipps in mehreren Stufen und eine aufklappbare Lösung.
- Das Spiel prüft das Ziel automatisch. Beispiel: „Dein ESP hat 3 Bots markiert“ oder „Bhop 10× hintereinander durch dein Skript“.
- Die Missionen bauen auf C++-Grundlagen auf (Variablen → Bedingungen → Schleifen → Funktionen → Structs → Zeiger → Klassen). Jede neue Idee wird direkt mit einem Cheat geübt. Beispiel: eine Schleife über alle Gegner, um das ESP zu bauen.
- Texte gibt es auf DE und EN. Code und Kommentare im Code sind auf Englisch.

## 8b. Fake-Desktop, Chat und Story (Rahmen des Spiels)

**Das Spiel startet wie eine virtuelle Maschine:** Nach einer lustigen Boot-Sequenz landet man auf dem Desktop eines eigenen Fake-Betriebssystems, Arbeitstitel **„HaxOS“**. Es ist an Windows angelehnt, hat aber einen eigenen Look und eigene Logos. Alles im Spiel ist eine „App“ auf diesem Desktop:

- **Das Spiel** (der Shooter): Man startet es per Doppelklick und es läuft im Vollbild.
- **Chat-App:** eine sichtbare Parodie eines Gamer-Chats. Sie hat Server, Kanäle und Direktnachrichten und sieht „gecrackt“ aus (falsche Badges, „Nitro Free Edition“-Gags). **Sie hat einen eigenen Namen, ein eigenes Logo und eigene Farben**, also weder den Namen noch das Logo des echten Produkts. Man soll die Anspielung erkennen, ohne dass es eine Kopie ist. Der Name ist noch offen (Vorschläge des Builders).
- **Code-Editor-App:** Hier schreibt man die Cheat-Module (Abschnitt 5).
- **Shop-App:** eine „Darknet“-Webseite im Fake-Browser, in der man Werkzeuge mit Credits kauft (Abschnitt 6).
- **Extras für Humor:** ein Papierkorb voller „gepasteter“ Cheats, Fake-Popups („Dein PC hat 47 Viren. Gar nicht wahr.“) und ein Einstellungen-Fenster.

**Story und Tutorial im Chat:**
1. Kurz nach dem Boot meldet sich eine Mentor-Figur im Chat und lädt einen auf den CheatLab-Server ein.
2. **Onboarding als Chat-Gespräch** mit Antwort-Buttons: „Schon mal einen Cheat benutzt?“, „Schon mal programmiert?“, „Kennst du C++?“. Die Antworten bestimmen, wie ausführlich die ersten Missionen erklären. Profis können die Grundlagen überspringen.
3. **Lustiger Einstieg:** Eine zwielichtige Figur verkauft einem „UltraHax 9000 PREMIUM“. Man spielt ein Match damit: Regenbogen-ESP, alles trifft, man fühlt sich wie ein Gott. Dann erscheint ein riesiger **BANNED**-Screen („Grund: Paste erkannt. lol.“). Der Mentor lacht: „Gekaufte Cheats kennt jedes Anti-Cheat. Bau deinen eigenen.“ → Mission 1.
4. Danach führt der Chat durch das Spiel. Figuren geben Missionen, Tipps, Spott und Lob, reagieren auf Erkennungen („bro, 180° Snap, ernsthaft?“) und kündigen neue Werkzeuge im Shop an.
- Die Story ist klein und lustig, keine riesige Handlung. Alle Texte gibt es auf DE und EN.
- **Später (Online-Phase):** Echte Kanäle für Spieler kommen dazu (Lobby-Chat, Code teilen), mit Moderation: Melden, Wortfilter, Blocken. Bis dahin schreiben nur NPCs.

## 9. Gameplay

- **Movement:** Source-artig mit Bhop, Air-Strafing und Crouch-Jump.
- **Modi zum Start:**
  1. **Team-Modus 5v5:** HvH-Gefühl. Das Ziel (Bombe oder Gebiete halten) wählt der Builder.
  2. **Battle Royale mit Bauen:** Die Zone wird kleiner, man baut Wände, Rampen und Böden. Eigener Stil, keine Kopie eines bestehenden Spiels.
  3. **Missionen (Lernmodus).**
- **Inhalt:** klein. 1–2 Karten für 5v5, 1 BR-Karte, 2–3 Waffen, Low-Poly.
- **Bots:** Schwierigkeit von leicht bis schwer einstellbar, Bots cheaten nie (sie haben keine Skripte). *(Claudes Entscheidung:)* Die höchsten Stufen haben **übermenschliche Reaktion und Treffergenauigkeit**, damit volle Lobbys nur mit guten Cheats zu schlagen sind. Das sorgt für das HvH-Gefühl offline.

## 10. Online (Phase 3)

- Server des Nutzers plus Self-Hosting.
- **Der Code der Spieler läuft auf dem Server**, im gleichen Interpreter mit gleichem Rechenbudget. Damit ist es fair und deterministisch, niemand kann den Interpreter manipulieren, und das AC sieht alles. Der Client schickt nur den Quellcode bzw. die Module beim Beitritt. Was Zeichen-Funktionen (ESP) brauchen, schickt der Server an den Client des Besitzers.
- **Rangliste:** Weltweit zählen nur Matches auf den offiziellen Servern des Nutzers. Daneben gibt es lokale Listen.

## 11. Phasen

0. **Spikes:** (a) Lässt sich ein C++-Teilmengen-Interpreter in s&box-C# schnell genug für `OnTick` mit 20+ Spielern bauen? (b) Bhop-Movement in s&box. (c) Standalone-Export testen und Freigabe bei Valve/Facepunch anfragen.
1. **Kern:** HaxOS-Desktop + Chat-App mit Intro-Story und Onboarding, Steam-Login + kleines Backend für den Fortschritt, eigene Modelle, Movement, 1v1-Aim-Map, Interpreter (Grundsprache), Editor, erste Werkzeuge (ESP, Aimbot, Bhop), Credits-Shop, AC-Stufen 1–2, erste Missionen.
2. **Ausbau:** 5v5 gegen Bots, Zeiger und virtueller Speicher, AC-Stufen 3–5, mehr Missionen, DE/EN.
3. **Online:** Server-Ausführung, echter Chat mit Moderation, Self-Hosting, Weltrangliste.
4. **Battle Royale mit Bauen.**
5. **Release** (s&box-Plattform und/oder Steam-Standalone, sonst Plan B).

Jede Phase bekommt einen eigenen Umsetzungsplan.

## 12. Entscheidungen des Nutzers

Alles im Spiel, wie Pure Skill, aber Cheats selbst programmieren · C++ · s&box · Credits schalten Werkzeuge frei · 5 AC-Stufen im Spiel, 4/5 todesschwer · Erwischt bedeutet Punktabzug + Anzeige · eigener C++-Interpreter · Einzelspieler zuerst, Online später · Bhop/HvH · Modi: 5v5, BR mit Bauen, Missionen · Missionen mit Aufgaben · Anfänger von null · AC-Regeln nach Stufe freischalten · DE+EN · Steam, Preis später · Plan B: anderer Launcher oder GitHub-Release, Closed Source · Weltrangliste nur von offiziellen Servern · Closed Source hat Vorrang → Standalone · Skripte dürfen lesen, steuern und (spät, teuer, riskant) eigene Werte schreiben · Fortschritt auf dem Server des Nutzers, gilt überall · AC-Stufen freischalten, dann frei wählen · Start als Fake-Desktop (VM-Gefühl) mit Parodie-Chat-App, Mentor-Onboarding per Chat (Cheat-/Programmier-Erfahrung abfragen), kleine lustige Story mit „gekaufter Cheat → gebannt“-Einstieg · Chat: erst NPCs, später echte Spieler · Fake-OS mit eigenem Gag-Namen · Team: Nutzer + Claude · Bots einstellbar · klein und simpel, eigene Maps später.

## 13. Entscheidungen von Claude (korrigierbar)

- Umfang der C++-Teilmenge (Abschnitt 5) und die Regel „was angenommen wird, ist echtes C++“.
- Einstiegs-Funktionen `OnTick`/`OnDraw`/`OnInput` und ein Rechenbudget pro Tick.
- Bequeme API vs. roher virtueller Speicher als Kernmechanik gegen das AC.
- Inhalt der AC-Stufen im Detail (Abschnitt 7).
- Online: Skripte laufen auf dem Server.
- Name „HaxOS“, Shop als Darknet-Webseite, Code-Editor als Desktop-App, Ablauf der Intro-Story.
- Chat-App: Parodie mit eigenem Namen und Logo, keine echten Marken.
- Verdachtswert mit gestuften Folgen beim Erwischtwerden.
- Bots auf höchster Stufe übermenschlich.
- Steam-Login, Backend ab Phase 1, offline spielbar ohne Credits.
- Reihenfolge der Phasen, BR zuletzt (am meisten Inhalt).
- Grafik und Sound werden gekauft oder sind kostenlos.

## 14. Dem Builder überlassen

Spielziel im 5v5, Waffenwerte, Kartenlayout, Bot-KI, Preise im Shop, genaues Layout des virtuellen Speichers, die Missionsliste im Detail, Interpreter-Interna (Bytecode o. ä.), UI-Design und das Netzwerk-Protokoll.
