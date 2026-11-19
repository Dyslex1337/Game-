# CheatLab C++ Interpreter (Kern) – Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eine eigenständige C#-Bibliothek, die eine Teilmenge von C++ („CheatLab C++“) kompiliert und in einer abgeschotteten VM mit Rechenbudget, simuliertem Speicher und freischaltbarer Spiel-API ausführt.

**Architecture:** Lexer → Parser (AST) → Binder (Typprüfung) → Compiler (Bytecode) → Stack-VM. Alle Skript-Daten (auch lokale Variablen) liegen in einem `VirtualMemory`, damit `&x` und Zeiger echte Adressen sind. Die Bibliothek kennt s&box nicht; das Spiel bindet sie später über `HostApi` an.

**Tech Stack:** C# / .NET 9 class library, xUnit, optional `g++` für den C++-Kompatibilitätstest.

**Spec:** `docs/superpowers/specs/2026-11-15-cheatlab-fps-design.md` (Abschnitte 5, 6, 7, 10)

**Einordnung:** Die Spec ist zu groß für einen Plan. Dies ist **Plan 1 von 6**; er braucht weder s&box noch den PC des Nutzers und enthält Spike (a) aus Phase 0. Folge-Pläne (eigene Dokumente, später):
2. Spikes auf dem PC des Nutzers: Bhop-Movement in s&box, Standalone-Export, Interpreter in s&box einbinden.
3. HaxOS-Desktop + Chat-App + Intro-Story/Onboarding.
4. Spiel-Kern: 1v1-Aim-Map, Code-Editor-App, erste Werkzeuge (ESP, Aimbot, Bhop).
5. Backend: Steam-Login, Fortschritt, Credits-Shop.
6. Anti-Cheat Stufen 1–2 + erste Missionen.

## Global Constraints

- Projekt: `engine/CheatLab.Script/` (Bibliothek), `engine/CheatLab.Script.Tests/` (xUnit), `engine/CheatLab.Script.Bench/` (Konsole). Keine Referenz auf s&box.
- **s&box-Whitelist-sicher:** kein `System.IO`, kein `System.Reflection(.Emit)`, kein `System.Threading`, kein `unsafe`, kein P/Invoke, kein `dynamic`.
- **Deterministisch** (Online läuft auf dem Server, Spec 10): `float` = C# `float` (32 bit), `int` = 32 bit mit Überlauf-Wraparound, keine Zufallszahlen, keine Uhrzeit, keine Abhängigkeit von Dictionary-Reihenfolge in Ausgaben.
- **Spieler-Code darf nie eine C#-Exception nach außen werfen.** Jeder Fehler wird `Diagnostic` (Kompilieren) oder `RuntimeError` (Ausführen).
- **Alle Meldungen DE + EN** (Spec 8), Bezeichner und Code-Kommentare Englisch.
- **„Was angenommen wird, ist echtes C++“** (Spec 5): jedes Beispielskript in den Tests muss auch mit `g++ -std=c++17 -fsyntax-only` gegen `engine/cpp-compat/game.h` durchgehen.
- Sprachumfang dieses Plans: `int float bool char void`, Arrays fester Größe, `struct` mit Feldern und Methoden, Zeiger, Referenzen, `const`, `if/else for while do break continue return`, Funktionen inkl. Rekursion, `#include "game.h"` (wird ignoriert), `ReadMemory<T>(addr)` / `WriteMemory<T>(addr, v)`. **Nicht** in diesem Plan: `std::string`, `std::vector`, `class`-Vererbung, eigene Templates, Exceptions.

## Review Focus

1. **Endlosschleife / tiefe Rekursion** → Skript wird nach Budget bzw. Stack-Limit gestoppt mit verständlicher Meldung, Host läuft weiter (Task 5).
2. **Dereferenzieren von `nullptr` oder Müll-Adressen** → `RuntimeError` „Zugriffsverletzung bei 0x…, Zeile N“, keine `IndexOutOfRangeException` (Task 6).
3. **Typische Anfängerfehler** (fehlendes `;`, Tippfehler im Namen, `if (x = 5)`) → Meldung mit Zeile + Vorschlag („Meintest du `health`?“, Warnung bei Zuweisung in Bedingung) (Tasks 3, 4).
4. **Schreiben in schreibgeschützten Speicher** (andere Spieler, Spec 6) → `RuntimeError` „Schreibschutz“, Wert bleibt unverändert (Task 6).
5. **Umlaute/Emoji in Kommentaren und Strings, leere Datei, sehr große Datei (1 MB)** → kein Absturz, Lexer-Diagnose statt Exception, Quelltext-Limit 256 KB mit Meldung (Task 2).

---

### Task 1: Projekt-Gerüst + Diagnosen/Meldungen

**Files:**
- Create: `engine/CheatLab.sln`, `engine/CheatLab.Script/CheatLab.Script.csproj`, `engine/CheatLab.Script.Tests/CheatLab.Script.Tests.csproj`
- Create: `engine/CheatLab.Script/Diagnostics/Diagnostic.cs`, `engine/CheatLab.Script/Diagnostics/Messages.cs`
- Test: `engine/CheatLab.Script.Tests/MessagesTests.cs`

**Interfaces:**
- Produces:
  - `enum Language { De, En }`
  - `enum Severity { Error, Warning }`
  - `enum DiagnosticCode { UnexpectedChar, UnterminatedString, SourceTooLarge, ExpectedToken, UnexpectedToken, UndeclaredName, TypeMismatch, NotAFunction, WrongArgCount, MissingReturn, AssignInCondition, ToolLocked, DuplicateName, NotAnLValue, BudgetExceeded, StackOverflow, AccessViolation, WriteProtected, DivisionByZero, HostError }`
  - `record Diagnostic(DiagnosticCode Code, Severity Severity, int Line, int Column, string[] Args)`
  - `static string Messages.Format(Diagnostic d, Language lang)` → `"Zeile {Line}: {text}"` / `"Line {Line}: {text}"`

- [ ] **Step 1:** .NET 9 SDK installieren falls nötig (`dotnet --version` ≥ 9), Solution + beide Projekte anlegen, Tests referenzieren Bibliothek. `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- [ ] **Step 2: Failing tests**

```csharp
[Fact] public void Formats_german_with_line() =>
    Assert.Equal("Zeile 3: Unbekannter Name 'helth'. Meintest du 'health'?",
        Messages.Format(new Diagnostic(DiagnosticCode.UndeclaredName, Severity.Error, 3, 5, new[]{"helth","health"}), Language.De));
[Fact] public void Formats_english() =>
    Assert.Equal("Line 3: Unknown name 'helth'. Did you mean 'health'?", /* same diagnostic */ ...);
[Fact] public void Every_code_has_both_languages() // iterate Enum.GetValues, Format must not throw nor return text containing "{"
```

- [ ] **Step 3:** Run `dotnet test engine` → FAIL (types missing).
- [ ] **Step 4:** Implement. Texte als zwei `switch`-Ausdrücke pro Sprache; `UndeclaredName` ohne zweites Arg lässt den „Meintest du“-Teil weg.
- [ ] **Step 5:** `dotnet test engine` → PASS.
- [ ] **Step 6:** Commit `feat(script): project skeleton and bilingual diagnostics`.

### Task 2: Lexer

**Files:**
- Create: `engine/CheatLab.Script/Lexing/Token.cs`, `engine/CheatLab.Script/Lexing/Lexer.cs`
- Test: `engine/CheatLab.Script.Tests/LexerTests.cs`

**Interfaces:**
- Produces:
  - `enum TokenKind { Identifier, IntLiteral, FloatLiteral, CharLiteral, StringLiteral, Keyword, Punct, Eof }`
  - `record Token(TokenKind Kind, string Text, int Line, int Column)`
  - `static LexResult Lexer.Lex(string source)`; `record LexResult(IReadOnlyList<Token> Tokens, IReadOnlyList<Diagnostic> Diagnostics)` — letzte Token immer `Eof`.
- Keywords: `int float bool char void struct const if else for while do break continue return true false nullptr`. Punct inkl. `-> :: ++ -- += -= *= /= == != <= >= && || << >>`.

- [ ] **Step 1: Failing tests**
  - `"int x = 0x10;"` → `Keyword int, Identifier x, Punct =, IntLiteral 0x10, Punct ;, Eof`; `0x10` behält Text, Wert wird im Parser bestimmt.
  - `"1.5f"` → `FloatLiteral`; `p->hp` → `Identifier, Punct ->, Identifier`.
  - `// ümläut 🎯\n x` und `/* … */` werden übersprungen, `x` hat `Line == 2`.
  - `#include "game.h"` erzeugt **keine** Tokens.
  - `"abc` (offen) → Diagnose `UnterminatedString`, kein Throw.
  - `"$"` → `UnexpectedChar`; leere Quelle → nur `Eof`.
  - Quelle > 256 KB → genau eine Diagnose `SourceTooLarge`, Tokens = `[Eof]`.
- [ ] **Step 2:** `dotnet test engine --filter LexerTests` → FAIL.
- [ ] **Step 3:** Implement (handgeschriebener Scanner, Spalten 1-basiert).
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5:** Commit `feat(script): lexer`.

### Task 3: Parser (AST)

**Files:**
- Create: `engine/CheatLab.Script/Syntax/Ast.cs`, `engine/CheatLab.Script/Syntax/Parser.cs`
- Test: `engine/CheatLab.Script.Tests/ParserTests.cs`

**Interfaces:**
- Consumes: `Lexer.Lex`, `Token`, `Diagnostic`
- Produces:
  - `static ParseResult Parser.Parse(IReadOnlyList<Token> tokens)`; `record ParseResult(CompilationUnit Unit, IReadOnlyList<Diagnostic> Diagnostics)`
  - AST-Records (alle mit `int Line`): `CompilationUnit(List<Decl>)`; `Decl` = `FuncDecl(TypeSyntax Ret, string Name, List<Param> Params, Block Body)` | `StructDecl(string Name, List<FieldDecl> Fields, List<FuncDecl> Methods)` | `GlobalVarDecl(VarDecl)`; `TypeSyntax(string Name, int PointerDepth, bool IsRef, bool IsConst, int? ArraySize, TypeSyntax? TemplateArg)`
  - Stmt: `Block, VarDecl(TypeSyntax, string Name, Expr? Init), ExprStmt, If, While, DoWhile, For, Return, Break, Continue`
  - Expr: `Literal, Name, Unary(op), Binary(op), Assign(op), Call(Expr Callee, List<Expr> Args, TypeSyntax? TemplateArg), Member(Expr, string, bool Arrow), Index, AddressOf, Deref, Conditional(cond ? a : b), Cast(TypeSyntax, Expr)` (C-Cast `(int)x` und `static_cast<int>(x)`)
- Präzedenz wie C++ (Zuweisung rechts-assoziativ, `||` < `&&` < Vergleich < `+` < `*` < unär < postfix).

- [ ] **Step 1: Failing tests**
  - `int add(int a, int b) { return a + b * 2; }` → `Binary(+, a, Binary(*, b, 2))`.
  - `struct Player { float hp; void Heal(float v) { hp += v; } };` → `StructDecl` mit 1 Feld, 1 Methode.
  - `Player* p = &me; p->hp = 5;` → `AddressOf`, `Member(Arrow=true)`.
  - `int h = ReadMemory<int>(base + 0x100);` → `Call` mit `TemplateArg.Name == "int"`.
  - `for (int i = 0; i < 10; i++) {}` parst.
  - **Fehlendes `;`**: `int x = 5\nint y = 6;` → Diagnose `ExpectedToken` mit Args `[";"]`, **Zeile 1**, Parser erholt sich und findet `y` (Recovery bis `;`/`}`).
  - Ein Fehler erzeugt höchstens eine Diagnose pro Anweisung (keine Fehlerlawine).
- [ ] **Step 2:** FAIL. **Step 3:** Recursive-descent implementieren. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat(script): parser`.

### Task 4: Binder / Typprüfung

**Files:**
- Create: `engine/CheatLab.Script/Semantics/ScriptType.cs`, `engine/CheatLab.Script/Semantics/Binder.cs`, `engine/CheatLab.Script/Semantics/BoundTree.cs`
- Test: `engine/CheatLab.Script.Tests/BinderTests.cs`

**Interfaces:**
- Consumes: AST aus Task 3; `HostApi` aus Task 7 — bis dahin ein leeres `HostApi` (Task 7 legt die Klasse an; hier zunächst `HostApi` als leere Klasse mit `TryGet(string name, out HostFunction f)` anlegen, die Task 7 ausbaut).
- Produces:
  - `abstract record ScriptType { int Size }`: `IntType(4) FloatType(4) BoolType(1) CharType(1) VoidType(0) PointerType(ScriptType Elem)(4) RefType(ScriptType Elem) ArrayType(ScriptType Elem, int Count) StructType(string Name, IReadOnlyList<Field> Fields)`; `record Field(string Name, ScriptType Type, int Offset)` — Felder auf 4 Byte ausgerichtet.
  - `static BindResult Binder.Bind(CompilationUnit unit, HostApi api)`; `record BindResult(BoundProgram? Program, IReadOnlyList<Diagnostic> Diagnostics)` — `Program` ist `null`, wenn ein `Error` vorliegt.
  - `BoundProgram` enthält gebundene Funktionen (Methoden als `Struct::Name` mit implizitem `this`-Zeiger), Globals, Structs; jeder gebundene Ausdruck kennt seinen `ScriptType`.
- Regeln: implizite Konvertierung `int↔float`, `bool↔int`, `nullptr→T*`; Zeigerarithmetik `T* + int` skaliert mit `sizeof(T)`; `int + pointer` erlaubt wie C++; Adress-Arithmetik auf `int` (für `base + 0x100`) erlaubt.

- [ ] **Step 1: Failing tests**
  - `int f() { return helth; }` mit Global `int health;` → `UndeclaredName` Args `["helth","health"]` (Vorschlag via Levenshtein ≤ 2).
  - `int f() { int x = 1; if (x = 5) {} return x; }` → genau eine `Warning AssignInCondition`, `Program != null`.
  - `int f() { }` → `MissingReturn`; `void f() {}` → ok.
  - `float* p = 5.0f;` → `TypeMismatch`.
  - `struct V { float x; float y; char c; };` → Offsets `0, 4, 8`, `Size == 12`.
  - `int f(int a) { return f(); }` → `WrongArgCount`.
  - `5 = x;` → `NotAnLValue`; doppelte Funktion → `DuplicateName`.
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat(script): binder and type checking`.

### Task 5: Compiler + VM (Skalare, Kontrollfluss, Budget)

**Files:**
- Create: `engine/CheatLab.Script/Compiling/OpCode.cs`, `engine/CheatLab.Script/Compiling/Compiler.cs`, `engine/CheatLab.Script/Runtime/Vm.cs`, `engine/CheatLab.Script/Runtime/RunResult.cs`
- Test: `engine/CheatLab.Script.Tests/VmTests.cs`, Helper `engine/CheatLab.Script.Tests/TestScript.cs`

**Interfaces:**
- Consumes: `BoundProgram`
- Produces:
  - `static ScriptModule Compiler.Compile(BoundProgram p)` — unveränderlich, darf von mehreren VMs geteilt werden.
  - `readonly struct Value { int I; float F; }` mit `Value.Int(int)`, `Value.Float(float)`.
  - `sealed class Vm(ScriptModule module, VirtualMemory memory, HostApi api)` (`VirtualMemory` legt Task 6 an; hier minimal: Stack-Region als `byte[]`).
  - `RunResult Vm.Call(string function, Value[] args, int instructionBudget)`; `record RunResult(bool Ok, Value Return, RuntimeError? Error, int InstructionsUsed)`; `record RuntimeError(DiagnosticCode Code, int Line, string[] Args)`.
  - Konstanten: `Vm.MaxCallDepth = 256`.
- Test-Helper: `TestScript.Run(string src, string fn = "main", int budget = 100_000, params Value[] args)` → `RunResult` (Lex→Parse→Bind→Compile→Vm), Assert dass keine Kompilierfehler.

- [ ] **Step 1: Failing tests**
  - `int main(){ return 2 + 3 * 4; }` → `Return.I == 14`.
  - `int main(){ int s=0; for(int i=1;i<=100;i++) s+=i; return s; }` → `5050`.
  - `int fib(int n){ return n<2?n:fib(n-1)+fib(n-2); }` → `fib(20) == 6765`.
  - `float main(){ return 7 / 2.0f; }` → `3.5f`; `int main(){ return 7/2; }` → `3`; `int` Überlauf `2147483647 + 1` → `-2147483648`.
  - **Endlosschleife** `while(true){}` mit Budget 10 000 → `Ok == false`, `Error.Code == BudgetExceeded`, `InstructionsUsed == 10000`.
  - **Rekursion** `int r(int n){ return r(n+1); }` → `StackOverflow`, kein C#-`StackOverflowException` (VM iterativ, kein C#-Rekursionsaufruf pro Skript-Call).
  - `1/0` → `DivisionByZero` mit korrekter Zeile.
  - Zwei Aufrufe derselben VM hintereinander: Globals bleiben erhalten (Zustand über Ticks, für `OnTick`).
- [ ] **Step 2:** FAIL. **Step 3:** Stack-VM implementieren; jede Instruktion trägt ihre Quellzeile (für Fehler). Budget = Anzahl ausgeführter Instruktionen. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat(script): bytecode compiler and VM with instruction budget`.

### Task 6: Virtueller Speicher, Zeiger, Structs, Arrays

**Files:**
- Create: `engine/CheatLab.Script/Runtime/VirtualMemory.cs`
- Modify: `Compiler.cs`, `Vm.cs` (Locals/Globals im `VirtualMemory`-Stack/Data-Bereich statt in C#-Slots)
- Test: `engine/CheatLab.Script.Tests/MemoryTests.cs`

**Interfaces:**
- Produces:
  - `enum Access { None, Read, ReadWrite }`
  - `sealed class VirtualMemory(int size)`: `void MapRegion(uint start, int length, Access access, string name)`; `bool TryReadInt32(uint addr, out int v)`, `TryReadSingle`, `TryReadByte`, `MemoryResult TryWriteInt32(uint addr, int v)` (+ Single, Byte); `enum MemoryResult { Ok, Unmapped, WriteProtected }`; `void HostWriteInt32(uint addr, int v)` / `HostWriteSingle` (Spiel schreibt ohne Schutzprüfung).
  - Feste Layout-Konstanten in `VirtualMemory`: `NullGuard = 0x0000_0000..0x0000_FFFF` (nie gemappt), `ScriptData = 0x0001_0000` (64 KB Globals, RW), `ScriptStack = 0x0002_0000` (64 KB, RW), `GameBase = 0x1000_0000` (vom Host gemappt).
- Verhalten: `&x` liefert die echte Adresse im Stack; `*p`, `p->f`, `a[i]` lesen/schreiben über `VirtualMemory`; Fehler → `RuntimeError(AccessViolation, line, ["0x…"])` bzw. `WriteProtected`.

- [ ] **Step 1: Failing tests**
  - `int main(){ int x=5; int* p=&x; *p=9; return x; }` → `9`.
  - `struct V{float x; float y;}; float main(){ V v; V* p=&v; p->y=2.5f; return v.y; }` → `2.5f`.
  - `int main(){ int a[4]; for(int i=0;i<4;i++) a[i]=i*i; int* p=a; return *(p+3); }` → `9`.
  - `void inc(int& r){ r++; } int main(){ int x=1; inc(x); return x; }` → `2`.
  - **nullptr**: `int main(){ int* p=nullptr; return *p; }` → `AccessViolation`, Args `["0x00000000"]`.
  - **Müll-Adresse** `*(int*)0x7FFFFFF0` → `AccessViolation`, kein C#-Exception.
  - **Schreibschutz**: Host mappt `0x1000_0000` als `Read`, schreibt dort `100` mit `HostWriteInt32`; Skript `WriteMemory<int>(0x10000000, 1)` → `WriteProtected`, danach liest Host weiter `100`.
  - `ReadMemory<float>(addr)` liest Host-Wert korrekt.
- [ ] **Step 2:** FAIL. **Step 3:** Implement (little-endian, Float über `BitConverter.SingleToInt32Bits`). **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat(script): virtual memory, pointers, structs and arrays`.

### Task 7: Host-API mit freischaltbaren Werkzeugen

**Files:**
- Create/Modify: `engine/CheatLab.Script/Runtime/HostApi.cs`
- Test: `engine/CheatLab.Script.Tests/HostApiTests.cs`

**Interfaces:**
- Produces:
  - `delegate Value HostCall(HostContext ctx, ReadOnlySpan<Value> args)`; `HostContext` gibt Zugriff auf `VirtualMemory` und die aktuelle Zeile.
  - `sealed record HostFunction(string Name, ScriptType Return, IReadOnlyList<ScriptType> Params, HostCall Impl, string? ToolId, int Cost)` — `Cost` = Instruktionen, die vom Budget abgezogen werden.
  - `sealed class HostApi`: `void Register(HostFunction f)`, `void Unlock(string toolId)`, `bool IsUnlocked(string toolId)`, `bool TryGet(string name, out HostFunction f)`, `event Action<string toolId,int line>? ToolUsed` (für das spätere Anti-Cheat Stufe 3: „bequeme API wird geloggt“, Spec 7).
  - Host-Structs: `void RegisterStruct(StructType t)` damit z. B. `Vec3` im Skript bekannt ist.
- Verhalten: Aufruf einer **gesperrten** Funktion → Kompilierfehler `ToolLocked` Args `[name]` („Werkzeug 'GetEnemies' ist noch gesperrt – kauf es im Shop.“). `ReadMemory`/`WriteMemory` sind normale Host-Funktionen mit `ToolId = "raw_memory"` und Template-Arg-Sonderbehandlung im Binder.

- [ ] **Step 1: Failing tests**
  - Registriere `int GetEnemyCount()` mit `ToolId="enemies"`, gesperrt → Skript `return GetEnemyCount();` → `ToolLocked`.
  - Nach `Unlock("enemies")` → Ergebnis des Delegaten; `ToolUsed` feuert mit `("enemies", 1)`.
  - `Cost = 50`, Budget 40 → `BudgetExceeded`.
  - Host-Funktion wirft C#-Exception → `RuntimeError(HostError)`, Host-Prozess läuft weiter.
  - `RegisterStruct(Vec3{x,y,z})` + Host-Funktion `Vec3 GetPos()` → Skript liest `GetPos().y`.
- [ ] **Step 2:** FAIL. **Step 3:** Implement. **Step 4:** PASS.
- [ ] **Step 5:** Commit `feat(script): host API with unlockable tools`.

### Task 8: Fassade, Beispiel-Cheats, C++-Kompatibilität, Benchmark (Spike a)

**Files:**
- Create: `engine/CheatLab.Script/ScriptHost.cs`
- Create: `engine/cpp-compat/game.h` (echte C++-Deklarationen der Test-API: `struct Vec3`, `int GetEnemyCount()`, `Vec3 GetEnemyPos(int)`, `void SetViewAngles(float,float)`, `bool IsOnGround()`, `void Jump()`, `template<class T> T ReadMemory(unsigned)`, `template<class T> void WriteMemory(unsigned, T)`)
- Create: `engine/CheatLab.Script.Tests/Samples/esp_count.cpp`, `aimbot_basic.cpp`, `bhop.cpp`, `raw_memory_health.cpp`
- Create: `engine/CheatLab.Script.Tests/SampleTests.cs`, `engine/CheatLab.Script.Tests/CppCompatTests.cs`
- Create: `engine/CheatLab.Script.Bench/Program.cs` (+ csproj)

**Interfaces:**
- Produces:
  - `static CompileResult ScriptHost.Compile(string source, HostApi api)`; `record CompileResult(ScriptModule? Module, IReadOnlyList<Diagnostic> Diagnostics)`
  - `ScriptInstance ScriptHost.Instantiate(ScriptModule m, VirtualMemory mem, HostApi api)`; `RunResult ScriptInstance.Tick(int budget)` ruft `void OnTick()` falls vorhanden.
  - `const int ScriptHost.DefaultTickBudget = 5_000`.

- [ ] **Step 1: Failing tests (SampleTests)** — Fake-Spiel im Test: 3 Gegner, Spieler-Struct bei `GameBase`.
  - `bhop.cpp`: `OnTick` springt nur wenn `IsOnGround()` → über 10 Ticks mit abwechselndem Boden genau 5 `Jump()`-Aufrufe.
  - `aimbot_basic.cpp`: `SetViewAngles` wird mit dem Winkel zum nächsten Gegner aufgerufen (Toleranz 0.01).
  - `raw_memory_health.cpp`: liest `health` über `ReadMemory<int>(GameBase + 0x100)` → gleich dem Host-Wert.
  - Alle Samples kompilieren ohne Diagnosen und bleiben unter `DefaultTickBudget`.
- [ ] **Step 2: CppCompatTests** — für jede Datei in `Samples/`: `g++ -std=c++17 -fsyntax-only -include ../cpp-compat/game.h <file>` Exit 0. Ist `g++` nicht installiert → Test als übersprungen melden (`Skip`), nicht als bestanden.
- [ ] **Step 3:** FAIL → implementieren → PASS (`dotnet test engine`).
- [ ] **Step 4: Benchmark (Spike a)** — `dotnet run -c Release --project engine/CheatLab.Script.Bench`: 20 Instanzen von `aimbot_basic.cpp` + einer Schleife auf volle 5 000 Instruktionen, 640 Ticks. Ausgabe: `avg ms per server tick (20 scripts)`. **Ziel: < 4 ms** (bei 64 Ticks/s ≈ 25 % eines Ticks). Ergebnis in `docs/superpowers/spikes/2026-interpreter-bench.md` notieren; liegt es darüber, stoppen und mit dem Nutzer über Budget/Tickrate reden, bevor Plan 4 startet.
- [ ] **Step 5:** Commit `feat(script): script host facade, sample cheats, compat check and benchmark`.
