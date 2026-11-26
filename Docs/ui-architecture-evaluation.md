# UI-Architektur von TRACER/VPET – Evaluation als Entscheidungsvorlage

**Zweck:** Diskussionsgrundlage für das Team-Meeting. Das Dokument ist eine Entscheidungshilfe und kein Beschluss.
**Code-Basis:** TRACER, Commit `d2f8e16`. Das ist der Stand, auf den das Submodul `VPET_Unity2/Assets/TRACER` im Fork zeigt. Alle Pfade unten sind relativ zum TRACER-Repo.
**Methode:** Ich habe nur Code gelesen. Nichts wurde ausgeführt oder getestet, und am Code wurde nichts geändert.

**Kennzeichnung**

- **[Code]**: direkt aus dem Code gelesen, mit Dateiverweis.
- **[Annahme]**: Vermutung oder Schlussfolgerung, die im Team geprüft werden sollte.
- Kapitel 2–6 enthalten Vorschläge. Die Marker zeigen dort, worauf sich ein Vorschlag stützt.

---

## 0. Kurzfassung

1. Das heutige UI-System ist gut für schnelles Prototyping per Code. Es ist schlecht für Design-Änderungen, weil **das Aussehen zweimal definiert ist (Prefab und Code) und der Code gewinnt**. [Code]
2. Die Beobachtungen aus der ersten Analyse treffen im Kern zu. Dazu kommen zwei Punkte: Es gibt bereits **zwei parallele Parameter-UI-Systeme**, und Rollen werden mit **zwei verschiedenen Logiken** geprüft. [Code]
3. Drei Varianten wurden betrachtet:
   - **A** „Registry + Theme“ (klein)
   - **B** „Prefab-First“ (groß)
   - **C** „Code-First mit optionalen Overrides“ (Hybrid)
4. **Tendenz: Variante C, in drei Stufen eingeführt.** Stufe 1 ist identisch mit Variante A und lohnt sich unabhängig davon, wie das Team entscheidet.
5. Gegen B als alleiniges System spricht ein konkreter Fall: Das Settings-Menü wird zur Laufzeit per Reflection aus den Settings-Klassen erzeugt und kann kein statisches Prefab sein. B bräuchte also zusätzlich einen Code-Weg und damit zwei Systeme. [Code: `Core/Managers/UIManager/UIManager.cs`, `CreateSettingsMenu` / `createMenuTreefromSettings`]

---

## 1. Ist-Zustand

### 1.1 Der Ist-Zustand in 10 Zeilen

1. **Datenmodell:** `MenuTree`/`MenuItem` ist ein reiner C#-Baum ohne Unity-Typen. Er enthält Elementtypen (`HSPLIT, VSPLIT, SPACE, TEXT, TEXTSECTION, TEXTBOX, PARAMETER, IMAGE`), Parameter, Icons als `byte[]` und Klick-Aktionen. Elemente haben **keine ID**. [Code: `Core/Menus/MenuTree.cs`]
2. **Ein Renderer für alles:** `MenuCreatorModule` lädt 11 feste Prefabs (`Resources.Load("Prefabs/...")`). Es baut das Menü mit einem großen `switch` über Elementtyp und Parametertyp. Ein **zweiter `switch`** aktualisiert die Werte. [Code: `Modules/UIManagerModules/MenuCreatorModule/Scripts/MenuCreatorModule.cs`, Z. 112–122, 201–572, 594–691]
3. **Aussehen im Code:** Fast jeder Fall setzt Farbe, Schrift und Schriftgröße aus einem globalen Farb-Asset (`DATA_VPET_Colors`) und überschreibt damit das Prefab. [Code: `MenuCreatorModule.cs`, z. B. Z. 245–247, 296–301; `Core/Managers/UIManager/VPETUISettings.cs`]
4. **Layout im Code:** `HSPLIT`/`VSPLIT` sind dasselbe leere Prefab (`MenuPanel`). Die LayoutGroup wird per Code mit festen Werten angehängt. Kinder ändern die LayoutGroup ihres Eltern-Elements (`TEXT`, `RPCParameter<int>`). [Code: `MenuCreatorModule.cs`, Z. 204–229, 237–240, 337–339]
5. **Prefab-Zugriff über Namen und Positionen:** Der Renderer greift über `Find("Panel")`, `Find("Panel_Menu")`, `FindDeepChild("Text"/"Button")` und `GetChild(0)` auf Prefab-Teile zu. Im Core und in den UI-Modulen gibt es rund 59 solcher Stellen. [Code: `MenuCreatorModule.cs`, Z. 161–172, 342; `UICreator2DModule.cs`, Z. 111–116; `DialogCreatorModule.cs`, Z. 114–117]
6. **Zweites, paralleles UI-System:** `UICreator2DModule` zeigt die Parameter des ausgewählten Objekts mit eigenen Prefabs (Spinner, ColorPicker, ListSelector) und einem eigenen `switch`. Es nutzt dafür aber schon ein **Adapter-Muster** (`Manipulator.Init(parameter, manager)`). [Code: `UICreator2DModule.cs`, Z. 326–386; `Scripts/Manipulator/Manipulator.cs`]
7. **Rollen:** Das Enum `UIManager.Roles` hat fünf Werte. Die Namensliste steht **zweimal** im `UIManager`. Menüs und Buttons prüfen „ist meine Rolle in der Liste“, Parameter dagegen prüfen eine **Rangfolge** (`_role >= activeRole`). Zusätzlich gibt es fest verdrahtete Prüfungen in Modulen. [Code: `UIManager.cs`, Z. 74, 290–326; `MenuSelectorCreatorModule.cs`, Z. 137–139; `Core/Parameters/ParameterObject.cs`, Z. 164; `IconCreatorModule.cs`, Z. 164 ff.; `EvaluationHelper.cs`, Z. 720 ff.]
8. **Immer nur ein Menü:** Beim Öffnen eines Menüs wird das vorherige zerstört. Das Menü wird jedes Mal komplett neu aufgebaut. [Code: `MenuCreatorModule.cs`, Z. 147–190]
9. **Input:** `UnityInputModule` entscheidet pro Geste, ob sie über 2D-UI liegt (`EventSystem.RaycastAll`). Danach sendet es getrennte Events wie `dragUIEvent` und `dragOtherEvent`. Die Kamera hört nur auf `*Other`, der Controller navigiert über Unitys `Selectable`. [Code: `UnityInputModule.cs`, Z. 1011–1135; `EvaluationHelper.cs`, Z. 462, 518; `CameraNavigationModule.cs`, Z. 218–221; `ControllerModule.cs`, Z. 836, 880]
10. **Engine-Grenze:** Der Core ist **heute nicht Unity-frei**. Das Core-asmdef referenziert `UnityEngine.UI`, `TextMeshPro` und `InputSystem`. `UIManager` nutzt `CanvasScaler` und `Resources`, Parameter nutzen `UnityEngine.Vector3`/`Color`. Nur das Menü-Datenmodell ist (fast) sauber, es hängt allein über `UIManager.Roles` am Unity-abhängigen `UIManager`. [Code: `Core/CoreAssembly.asmdef`; `UIManager.cs`, Z. 288, 389; `Core/Parameters/Parameter.cs`, Z. 36, 58–69]

Positiv: Module werden per Reflection automatisch gefunden, eine zentrale Liste muss niemand pflegen. Dieses Muster lässt sich für eine Element-Registry wiederverwenden. [Code: `Core/Interfaces/ManagerInterface.cs`, Z. 101–113]

### 1.2 Prüfung der Vorab-Beobachtungen

| Beobachtung | Ergebnis | Beleg |
|---|---|---|
| Ein neues Element erfordert einen Enum-Wert, mehrere Switch-Cases, MenuTree-Overloads und ein Prefab | **Bestätigt.** Ein neuer Parametertyp berührt zusätzlich `ParameterType` und die Typliste in `Parameter.cs` sowie den Switch in `UICreator2DModule`. Vermutlich ist er auch Teil des Netzwerkformats (`ParameterType : byte`) [Annahme]. | [Code] `MenuTree.cs` (Overload `Add(string, bool)` nur für `TEXTSECTION`), `MenuCreatorModule.cs`, `Parameter.cs`, Z. 58–74 |
| Eine abweichende Optik für einen Fall kann andere Menüs brechen | **Bestätigt.** Gründe: geteilte Prefabs (`MenuPanel` für drei Typen, ein `MenuButton` für alle Buttons), Kinder ändern das Layout der Eltern, und Code-Farben überschreiben Prefab-Farben. | [Code] `MenuCreatorModule.cs`, Z. 203, 220, 232, 237–240, 337–339 |
| Prefabs werden über Namen oder Index durchsucht | **Bestätigt**, besonders in `UICreator2DModule` (verschachteltes `GetChild`). Im Menü-Renderer selbst sind es nur wenige Stellen. | [Code] siehe Zeile 5 oben |

### 1.3 Die wichtigsten Ursachen

1. **Der Renderer kennt jeden Einzelfall.** Jede Erweiterung ist eine Änderung an zentralen Switches. [Code]
2. **Das Aussehen hat zwei Besitzer.** Das Prefab und der Code setzen beide Farbe und Schrift, und der Code läuft zuletzt. Deshalb wirken Änderungen der Designerin nicht. [Code]
3. **Layout ist Code-Logik statt Prefab-Eigenschaft.** Wer das Layout ändern will, muss C# ändern. [Code]
4. **Zwischen Prefab und Code gibt es keinen ausdrücklichen Vertrag.** Es gibt nur implizite Namen und Positionen in der Hierarchie. Benennt die Designerin ein Objekt um, bricht das erst zur Laufzeit, oft mit einer NullReference. [Code + Annahme zur Fehlerart]
5. **Elemente haben keine Identität.** Ohne ID lässt sich „genau dieser Logout-Button“ nicht gezielt anders gestalten. [Code]
6. **Rollen sind verteilt und widersprüchlich.** Listen-Logik und Rangfolge-Logik stehen nebeneinander, an mehreren Stellen. [Code]

### 1.4 Nebenbefunde (nicht Kern der Frage, aber relevant)

- **Wahrscheinlicher Bug:** `MenuCreatorModule` registriert Updates nur für den *ersten* Parameter eines `ParameterObject`. Weitere Parameter desselben Objekts im selben Menü werden vermutlich nicht aktualisiert, wenn sich ihr Wert von außen ändert. [Code: Z. 576–581; die Wirkung ist eine Annahme, nicht getestet]
- **Mehrdeutige Ressourcen:** Alle `Resources`-Ordner teilen sich einen gemeinsamen Namensraum. Es gibt schon doppelte Pfade (`Images/button_frame_BG.png`, `Images/button_network.png`), und welche Datei geladen wird, ist dann nicht eindeutig. Für Skins/Paletten heißt das: Namen brauchen einen Präfix pro Modul. [Code: Dateisuche; die Wirkung ist eine Annahme]
- **Hover ist vorbereitet, aber nicht umgesetzt:** `GenericActionEnum.ONHOVER` existiert, ausgewertet wird nur `ONCLICK`. [Code: `MenuTree.cs`, Z. 237; `MenuCreatorModule.cs`, Z. 212–216; `GenericMenuClickListener.cs`]
- **Parameter haben keinen Wertebereich (min/max).** [Code: `Parameter.cs`, kein entsprechendes Feld gefunden] Das ist wichtig für Slider (Szenario 3).

---

## 2. Architekturvarianten

Alle drei Varianten teilen zwei Grundregeln aus der Team-Diskussion:

- **Das Prefab bestimmt Aussehen und Layout, der Code bestimmt den Inhalt.** Konkret: Der Code setzt keine Farben, Schriften oder Layout-Werte mehr.
- **Prefab-Teile werden über Verknüpfungen innerhalb des Prefabs angesprochen, nicht über `Find`/`GetChild`.** Dafür bekommt jedes Element-Prefab ein kleines Adapter-Script mit Inspector-Feldern. Solche Verknüpfungen sind auch mit `Resources` möglich. [Code: Rahmenbedingung aus der Aufgabe, gestützt durch das bestehende `Manipulator`-Muster]

Ein Adapter in wenigen Zeilen (Skizze):

```csharp
public class ToggleView : ElementView {          // liegt auf dem Toggle-Prefab
    [SerializeField] Toggle toggle;               // Verknüpfung innerhalb des Prefabs
    public override void Bind(MenuItem item) {
        var p = (Parameter<bool>)item.Parameter;
        toggle.isOn = p.value;
        toggle.onValueChanged.AddListener(p.setValue);
        p.hasChanged += (_, v) => toggle.isOn = v; // Update hier – kein zweiter Switch
    }
}
```

Erzeugen und Aktualisieren liegen damit an **einer** Stelle pro Elementtyp und nicht mehr in zwei großen Switches.

### Variante A – „Registry + Theme“ (konservativ, klein)

```
 Modul (Code) ──► MenuTree  (Inhalt + grobes Layout wie heute)
                     │
 Renderer ── fragt ──► Registry:  Elementtyp/Parametertyp → Element-Prefab
                     │            (EINE globale Tabelle, als Asset)
                     ▼
 Element-Prefab + Adapter (Bind)  →  Aussehen kommt nur aus dem Prefab
```

**Leitprinzip:**

1. Der Code beschreibt Menüs genau wie heute mit `MenuTree`.
2. Statt Switches hat der Renderer eine Tabelle „Typ → Prefab“. Jedes Prefab bringt seinen eigenen Adapter mit.
3. Farben und Schriften stehen nur noch im Prefab, wahlweise über Theme-Bausteine wie „Farbe: ButtonBG“, die die Designerin im Prefab wählt.

**Ziele:**

| Ziel | Bewertung |
|---|---|
| 1 Prototyping per Code | ✔ unverändert schnell |
| 2 Kein SPOF, wenig Wartung | ✔ Switches fallen weg. Neue Elementtypen werden automatisch gefunden, wie heute die Module. Es gibt nur wenige String-IDs. |
| 3 Design ohne Code | ◐ Globale Stiländerungen gehen ohne Code (alle Buttons, alle Listen). **Einzelfälle gehen nicht** („nur der Logout-Button rund und schwarz“), weil Elemente keine Identität haben. Menü-Layouts bleiben Code (`HSPLIT`/`VSPLIT`). |

### Variante B – „Prefab-First“ (Menü = Prefab, Code liefert Daten)

```
 Designerin:  Menü-Prefab "SceneBrowser"
              ├─ statische Elemente (Titel, Deko, fester Logout-Button)
              ├─ Content-Root  [content: "sceneList"]
              ├─ Button        [command: "logout"]
              └─ Rollen-Flags  [sichtbar für: Scout, Architekt]
                     ▲
 Bindeschicht: sucht IDs im Prefab und verbindet sie mit …
                     │
 Entwickler:  Modul liefert Daten und Aktionen unter IDs
              "sceneList" → Liste von Datensätzen,  "logout" → Aktion
```

**Leitprinzip:**

1. Jedes Menü ist ein Prefab der Designerin, das Layout gehört vollständig ihr.
2. Der Code liefert nur Daten und Aktionen unter vereinbarten Namen.
3. Ohne Prefab gibt es kein Menü.

**Ziele:**

| Ziel | Bewertung |
|---|---|
| 1 Prototyping per Code | ✘ Ein neues Menü braucht zuerst ein Prefab. Das Settings-Menü wird aus Reflection erzeugt und braucht ohnehin einen Code-Weg. [Code] Damit gibt es **zwei Systeme**. |
| 2 Kein SPOF, wenig Wartung | ◐ Es gibt keine zentralen Switches, dafür **viele String-IDs** (Menü, Content, Command, Rolle) in Prefabs. Fehler zeigen sich erst zur Laufzeit, es sei denn, es gibt Editor-Validierung. Rollen-Flags im Prefab verteilen Rechte-Logik in Assets, die Entwickler selten ansehen. |
| 3 Design ohne Code | ✔ maximal: Layout, Einzelstile und Listen-Optik liegen komplett bei der Designerin |

### Variante C – „Code-First mit optionalen Overrides“ (Hybrid)

```
 Modul (Code): MenuTree  + optionale Schlüssel  (Menü-ID, Element-ID)
                  │
 Renderer: für jedes Element "Welches Prefab?" – feste Suchreihenfolge:
     1. Override für genau dieses Element       (Skin des Menüs, per Element-ID)
     2. Palette dieses Menüs                    (per Typ, z. B. INT → Slider)
     3. globale Standard-Palette                (per Typ)  = heutige Optik
     4. eingebauter Default-Look                ← funktioniert immer
                  │
 Element-Prefab + Adapter (Bind)  →  Aussehen nur aus dem Prefab

 optional pro Menü: Skin-Prefab mit Slots  [slot: "header"], [slot: "list"] …
     Elemente mit passender ID landen im Slot. Ohne passenden Slot kommen sie
     in den Standard-Slot (nichts geht verloren) und es gibt eine Warnung.
```

**Leitprinzip:**

1. Ein Menü funktioniert immer allein aus Code, mit Default-Look.
2. Die Designerin kann jede Stufe der Suchreihenfolge gezielt überschreiben, ohne Code anzufassen.
3. Code und Prefab sind über wenige Schlüssel verbunden. Diese stehen im Code als Konstanten und werden beim Aufbau geprüft.

**Ziele:**

| Ziel | Bewertung |
|---|---|
| 1 Prototyping per Code | ✔ unverändert. IDs sind optional und erst nötig, wenn gestaltet wird. |
| 2 Kein SPOF, wenig Wartung | ✔/◐ Der Renderer bleibt eine zentrale Stelle. Er enthält aber keine Einzelfall-Logik mehr und ändert sich deshalb selten. Ein kaputtes Skin fällt auf Stufe 3/4 zurück, statt das Menü zu brechen. Es gibt nur **ein** System, weil alte Menüs ohne Skin genauso dargestellt werden wie heute. String-IDs nur dort, wo gestaltet wird. |
| 3 Design ohne Code | ✔ für Layout (Skin/Slots), Einzelstile (Element-Override) und Listen-Optik (Palette/Override). Code ist nötig, sobald **neue Daten oder neues Verhalten** gebraucht werden (siehe Szenarien). |

**Bezug zu den Team-Ideen:**

- „Prefab = Aussehen, Code = Inhalt“ ist in allen Varianten umgesetzt.
- „Palette pro Menü mit Fallback + Adapter `Bind`“ entspricht C, Stufen 2–4.
- „Benannte Content-Roots“ entspricht den Slots in C, allerdings **optional** und nicht Pflicht wie in B.
- „Rollen als Flags im Prefab“ empfehle ich nur für *Optik*-Unterschiede, nicht für Sichtbarkeit von Funktionen (siehe Szenario 2).
- Die „konservative Variante mit Skins + Registry“ ist im Grunde C ohne Slots.

---

## 3. Szenarien-Matrix

Spaltenlegende:

- **Code?**: muss Code geändert werden (ja/nein/optional)
- **Wer**: D = Designerin, E = Entwickler
- **Vertrag**: was vereinbart sein muss und wo es steht
- **Früh erkennen**: wie eine Abweichung auffällt

Gemeinsamer Vorschlag für alle Varianten:

- IDs stehen als **Konstanten im Code**, eine kleine Klasse pro Modul.
- Im Prefab wählt man sie aus einer **Dropdown-Liste**, die ein kleines Editor-Script aus diesen Konstanten füllt. So gibt es keine Tippfehler.
- Ein Menüpunkt **„UI validieren“** im Editor prüft alle Skins.
- Zur Laufzeit gibt es **Warnungen** für unbekannte IDs.

### Szenario 1 – SceneBrowser mit Liste aus Designer-Prefabs

Der **minimale Vertrag** ist ein Datensatz mit benannten Feldern und benannten Aktionen. Er lebt im Core und kommt ohne Unity-Typen aus:

```csharp
var scene = new UiRecord("scene")
    .Set("title", s.name).Set("thumbnail", s.previewPng)   // byte[], wie heute bei IMAGE
    .Set("info", s.location).Set("details", s.longInfo)    // "details" z. B. für Hover
    .Action("open", () => LoadScene(s.path));
```

Auf der Prefab-Seite markiert die Designerin Text, Bild und Button im Karten-Prefab mit „Feld: title“, „Feld: thumbnail“ und „Aktion: open“. Felder, die sie nicht zeigen will, lässt sie weg.

**Besteht das Listen-Prefab aus Prefabs?** Ja, mit Unitys Nested Prefabs. Der menüspezifische Button ist einfach ein fester Teil des Karten-Prefabs. Die Designerin legt ihn fest, er wird nicht dynamisch getauscht, und die Markierung „Aktion: open“ verbindet ihn. Die Registry ist dafür nicht nötig. [Annahme: Unity-Standardfunktion, im Projekt noch nicht genutzt]

**Hover:** Die Karte hat einen Bereich „bei Hover zeigen“, der auf ein weiteres Prefab oder ein Kind-Objekt zeigt. Dieses Prefab nutzt dieselben Felder (z. B. „details“). Auf Touch-Geräten gibt es kein Hover. Hier wären Halten (`holdUIEvent` gibt es [Code]) oder Antippen zum Aufklappen sinnvoll. [Annahme zu Zielgeräten → offene Frage]

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja, auch für jede Optik-Änderung der Karte | E (D liefert nur das globale Listen-Prefab) | Typ → Prefab in der Registry. Feldnamen gibt es nicht, der Adapter ist fest für diese Karte gebaut. | Compiler, Adapter-Felder im Inspector |
| B | ja, einmal (Daten + Aktion), danach nein | E: Datensatz, D: Menü- und Karten-Prefab | Content-ID „sceneList“, Feld- und Aktionsnamen. Steht im Code (Konstanten) und im Prefab. | Editor-Validierung, Laufzeitwarnung „Feld fehlt“ |
| C | ja, einmal (Daten + Aktion), danach nein | E: Datensatz, D: Karte + optional Menü-Skin | wie B, zusätzlich Menü-ID. Ohne Skin erscheint die Liste im Default-Look. | wie B. Fehlt ein Skin, gibt es den Default statt eines Fehlers. |

### Szenario 2 – Neue Rolle (Scene-Scout, Architekt, Drohnenpilot)

Ist-Zustand: Eine neue Rolle braucht heute einen Enum-Wert, Einträge in zwei Namenslisten und Anpassungen an mehreren festen Prüfungen. [Code] Dazu kommt eine **Falle in der Rangfolge-Logik**: Eine Rolle, die hinten im Enum angehängt wird, sieht nach `_role >= activeRole` **keine** Parameter mit der Standardrolle `VIEWER` mehr. [Code: `ParameterObject.cs`, Z. 164; `Parameter.cs`, Z. 107. Die Wirkung ist abgeleitet, nicht getestet]

**Kleinster Weg (Vorschlag):**

1. Rollen werden zu **Daten**, also eine Liste von Namen an *einer* Stelle.
2. Sichtbarkeit wird als **Menge** geprüft („sichtbar für: Scout, Architekt“) statt als Rangfolge, an einer gemeinsamen Stelle.
3. *Was* eine Rolle sieht oder darf, legt der Code fest (`MenuTree.roles` gibt es schon [Code]; neu wären Rollen pro Element).
4. *Wie* es für eine Rolle aussieht, darf das Prefab variieren, z. B. mit einem Rollen-Flag auf Deko-Teilen oder einem eigenen Skin pro Rolle.

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja | E | Rollenliste im Code | Compiler |
| B | ja, für die Rolle selbst. Danach kann D Sichtbarkeit über Flags allein steuern. | E + D | Rollennamen im Code **und** als Flags in vielen Prefabs | Editor-Validierung nötig. Sonst fallen vergessene Flags nicht auf, und Rechte-Logik wird unsichtbar. |
| C | ja, einmal (Rolle anlegen und festlegen, welche Menüs sie sieht). Optik pro Rolle: nein. | E für das *Was*, D für das *Wie* | Rollenliste im Code/Config; Rollen-Flags nur für Optik im Prefab | Compiler + Validierung „unbekannter Rollenname im Prefab“ |

**Kann es die Designerin allein?** Nein. Eine neue Rolle mit anderen *Funktionen* ist in jeder Variante eine Entwickler-Entscheidung. Allein machbar ist nur eine andere *Optik* für eine bestehende Rolle.

### Szenario 3 – INT-Parameter in einem Menü als Slider

Ein Slider braucht einen Wertebereich, den Parameter heute nicht haben [Code]. Dieser Wertebereich muss **einmal per Code** ergänzt werden, z. B. `.Add(param).Range(0, 100)` oder als optionale Eigenschaft am Parameter. Das ist eine Datenfrage, keine Designfrage.

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja: Bereich + Darstellungshinweis „slider“ am Element (die Palette ist nur global) | E + D (Slider-Prefab) | Name des Hinweises im Code und in der Registry | Laufzeitwarnung „kein Prefab für Hinweis slider“ |
| B | ja: Bereich. Danach legt D den Slider ins Menü-Prefab. | E + D | Parameter-/Content-ID | Validierung |
| C | ja: Bereich (einmalig). Danach setzt D in der Palette des Menüs „INT → Slider“ oder überschreibt nur dieses Element. | E + D | Menü-ID bzw. Element-ID | Validierung. Ohne Bereich: Warnung und Zahlenfeld als Fallback. |

### Szenario 4 – Popup zum Editieren eines Zahlenfelds

Ist-Zustand: Es ist immer nur ein Menü offen. Das Dialog-System kann nur Meldungen und Fortschritt anzeigen, und das über Namenssuche. [Code: `MenuCreatorModule.cs`, Z. 148; `DialogCreatorModule.cs`, Z. 114–123]

Vorschlag:

- Ein Entwickler baut **einmal** einen „Popup-Host“, also eine Ebene über dem Menü, die *ein* Element-Prefab zeigt, gebunden an denselben Parameter.
- Dazu kommt eine Komponente „öffnet Popup mit Prefab X“.
- Danach entscheidet die Designerin pro Menü oder Element, ob ein Zahlenfeld direkt editiert wird oder ein Popup öffnet.

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja (Popup-Host + Popup-Verhalten fest im Zahlenfeld-Adapter, also global für alle) | E | – | Compiler |
| B | ja (Popup-Host einmalig), danach nein | E, dann D | Popup-Prefab-Referenz im Menü-Prefab | Inspector (fehlende Referenz) |
| C | ja (Popup-Host einmalig), danach nein | E, dann D | Zahlenfeld-Override mit Komponente „öffnet Popup“; Verknüpfung innerhalb des Prefabs | Inspector. Fehlt das Popup-Prefab, wird direkt editiert und es gibt eine Warnung. |

### Szenario 5 – Color-Picker als kreisrundes Popup

Ist-Zustand:

- Im Menü wird `COLOR` heute als **vier Zahlenfelder** dargestellt. [Code: `MenuCreatorModule.cs`, Z. 498–529]
- Einen Farbwähler mit Shader gibt es schon im zweiten System (`ColorSelect`, ein `Manipulator` mit `Init(param)`). [Code: `UICreator2DModule/Scripts/ColorSelect.cs`]
- Ob er rund funktioniert, hängt an seiner Positions-Mathematik. [Annahme: die Berechnung geht von einem Rechteck aus]

Ehrlich gesagt: Die **runde Form als Optik** ist Designerinnen-Arbeit (Sprite/Shader/Prefab). Eine **neue Interaktion** (Winkel = Farbton, Abstand = Sättigung) ist Entwickler-Arbeit.

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja (Adapter für Kreis-Interaktion + Popup) | E + D (Optik) | Registry-Eintrag COLOR | Compiler |
| B | ja (Interaktion), Popup-Platzierung durch D | E + D | Popup-Referenz im Prefab | Inspector |
| C | ja (Interaktion einmalig + Popup-Host aus Szenario 4), danach nein | E + D | Palette: COLOR → Kreis-Picker (global oder pro Menü) | Inspector + Validierung |

### Szenario 6 – Verschiebbares Floating-Menu

Ist-Zustand:

- Ein Ziehen, das über 2D-UI beginnt, wird als `dragUIEvent` gemeldet, und die Kamera reagiert darauf nicht. Das ist eine gute Ausgangslage. [Code: `UnityInputModule.cs`, Z. 1062–1076; `CameraNavigationModule.cs`, Z. 218]
- Es kann aber nur **ein** Menü gleichzeitig offen sein. [Code]

Vorschlag:

- Eine **Komponente „Verschiebbar“** (einmal Code), die die Designerin auf jeden Griff in einem Prefab legen kann, z. B. auf die Titelleiste.
- Sie hört auf `dragUIEvent` (oder Unitys Drag-Events) und bewegt ihr Menü.
- Soll das Floating-Menu *neben* einem anderen Menü offen sein, muss der Renderer mehrere Menüs gleichzeitig verwalten. Das ist Code.
- Controller-Bedienung (Verschieben ohne Touch) ist eine eigene Frage. [Annahme → offene Frage]

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja (Komponente; mehrere Menüs gleichzeitig: ja) | E | – | – |
| B | ja (Komponente einmalig), danach D | E, dann D | Komponente im Menü-Prefab | Inspector |
| C | ja (Komponente einmalig; ggf. mehrere Menüs), danach D | E, dann D | Komponente im Menü-Skin. Ohne Skin: Default-Menü mit Griff. | Inspector |

### Szenario 7 – Neuer Typ „Mesh-Objekt“ (Preview, Name, Größe …)

Vorschlag: Das Mesh-Objekt bekommt **keinen** neuen `ParameterType`. Dieser Typ ist ein Byte-Enum im Parameter-System und vermutlich Teil des Netzwerkformats. [Code: `Parameter.cs`, Z. 74; Netzwerkbezug ist Annahme] Stattdessen ist es ein **Datensatz** wie die Szene in Szenario 1: Der Code liefert alle Felder (Preview, Name, Größe, Polygone …), und die Designerin entscheidet, welche sie zeigt.

Ehrlich gesagt: **Code ist zwingend**, um die Daten zu erzeugen, vor allem das Preview-Bild (Rendern/Thumbnail). Code ist auch nötig, wenn später ein neues Feld gezeigt werden soll, das es noch nicht gibt.

| Variante | Code? | Wer | Vertrag & wo | Früh erkennen |
|---|---|---|---|---|
| A | ja (neuer Elementtyp + Adapter, jede Optik-Variante = Code) | E | Registry-Eintrag | Compiler |
| B | ja (Datensatz), danach D | E + D | Feldnamen im Code (Konstanten) + Prefab | Validierung „Feld nicht geliefert“ |
| C | ja (Datensatz), danach D | E + D | wie B. Ohne Prefab: generische Default-Karte (Name/Wert-Liste). | wie B |

### Szenario 8 – Reines Prototyping an einem Nachmittag

| Variante | Code? | Wer | Weg | Vom Prototyp zum gestalteten Menü |
|---|---|---|---|---|
| A | ja (nur Code) | E | `MenuTree` schreiben wie heute | Nur globaler Stil. Für eine eigene Optik muss E einen neuen Elementtyp oder Adapter bauen. |
| B | ja + Prefab | E (+ D) | Ohne Prefab kein Menü. Es sei denn, es gibt einen Auto-Fallback, und dann ist es schon Variante C. | D baut das Menü-Prefab, E stellt Daten auf IDs um. |
| C | ja (nur Code) | E | `MenuTree` schreiben wie heute, sofort im Default-Look | E vergibt **eine** Menü-ID und bei Bedarf Element-IDs (wenige Zeilen). D legt Skin/Overrides an. Die Logik bleibt unverändert. |

---

## 4. Vergleichstabelle

| Kriterium | A Registry + Theme | B Prefab-First | C Hybrid (Code-First + Overrides) |
|---|---|---|---|
| Einführungsaufwand | niedrig | hoch | mittel (stufenweise) |
| Designer-Freiheit | niedrig–mittel (nur globaler Stil) | hoch | hoch (wo ein Skin existiert), sonst Default |
| Prototyping-Tempo | hoch (wie heute) | niedrig (Prefab zuerst) | hoch (wie heute) |
| SPOF-/Wartungsrisiko | niedrig (Switches weg, wenige IDs) | mittel–hoch (viele String-IDs, zwei Wege wegen Settings-Menü) | niedrig–mittel (zentraler, aber generischer Renderer; IDs nur wo gestaltet; Fallback statt Bruch) |
| Engine-Agnostik | gut (Inhalt bleibt in `MenuTree`) | mittel (Menüstruktur wandert in Unity-Prefabs; ein zweites Backend müsste alle Menüs neu bauen) [Annahme] | gut (Inhalt + IDs + Datensätze reines C#; nur Darstellung in Unity) |
| Migrationsaufwand | niedrig (bestehende Menüs laufen unverändert) | hoch (jedes Menü als Prefab neu) | niedrig (bestehende Menüs laufen unverändert, Gestaltung nach Bedarf) |

---

## 5. Grobe Aufwandsschätzung

[Annahme] Es handelt sich um Personentage (PT) für eine Entwicklerin bzw. einen Entwickler mit Unity-Erfahrung, **ohne** Designarbeit und ohne Puffer für Unbekanntes. Die Zahlen sind Größenordnungen für die Diskussion, keine Zusage.

| Paket | PT |
|---|---|
| **A** gesamt: Registry + Adapter für ~12 bestehende Elementtypen (4–5), Farben/Schriften aus dem Code in Prefabs/Theme-Bausteine (2–3), `Find`/`GetChild` im Menü-Renderer ersetzen (1), Regressionstest aller Menüs (2) | **8–12** |
| **B** gesamt: Bindeschicht, Slots, Commands, Rollen-Flags (8–10), Editor-Validierung (3–5), alle bestehenden Menüs als Prefab nachbauen + Code-Weg für das Settings-Menü (10–20) | **25–40** (plus viel Designarbeit) |
| **C** Stufe 1 = A | 8–12 |
| **C** Stufe 2: Menü-/Element-IDs, Palette pro Menü, Override-Kette, Validierung + ID-Dropdown | 5–8 |
| **C** Stufe 3: Datensätze für Listen, Slots, Hover-Bereich; **SceneBrowser als Pilot** | 8–12 |
| **C** gesamt | **21–32** (jede Stufe ist für sich nutzbar) |
| Zusatz je Szenario (alle Varianten): Popup-Host 2–3 · Verschiebbar-Komponente 1–2 · Wertebereich für Slider 1–2 · Kreis-Farbwähler-Interaktion 3–5 · Rollen als Daten/Mengen 3–5 (berührt Core und Selektion) | je nach Bedarf |

---

## 6. Empfehlung (Entscheidungshilfe)

**Tendenz: Variante C, eingeführt in drei Stufen.**

1. **Stufe 1 (entspricht A):** Die Switches werden durch eine Registry mit Adaptern ersetzt, das Aussehen kommt aus dem Code heraus. Das bringt sofort Nutzen, bricht nichts und ist für jede spätere Entscheidung die richtige Basis.
2. **Stufe 2:** Optionale IDs, Paletten pro Menü und Element-Overrides. Ab hier kann die Designerin Einzelfälle ohne Code gestalten („Logout rund und schwarz“).
3. **Stufe 3:** Datensätze und Slots, **pilotiert am SceneBrowser**. Der ist ohnehin ein Mockup [Code: `SceneBrowserModule.cs`, `@note mockup implementation`] und deckt Liste, Karte, festen Button und Hover ab.

**Gründe:**

- C erfüllt Ziel 1 so gut wie heute und Ziel 3 so gut wie B.
- Bei Ziel 2 vermeidet C das Hauptrisiko von B: zwei Systeme und viele IDs.
- Bestehende Menüs laufen unverändert weiter. Das zählt besonders für das reflection-basierte Settings-Menü. [Code]

**Bewusst nicht empfohlen (noch nicht):**

- Das zweite UI-System (`UICreator2DModule`) jetzt umzubauen. Es nutzt schon Adapter, und die Adapter-Schnittstelle sollte so gestaltet werden, dass es später auf dieselbe Registry umziehen *kann*.
- Weitere Architektur auf Vorrat, etwa ein Datenformat für Menüs in JSON.

### Offene Fragen ans Team

1. **Wo soll die Arbeit liegen?** Das UI-System liegt im **TRACER-Repo** (Submodul, `FilmakademieRnd/TRACER`), nicht im VPET-Fork. Werden Änderungen upstream eingebracht, oder gibt es einen eigenen TRACER-Fork bzw. Branch? [Code: `.gitmodules`]
2. **Engine-Agnostik:** Ist ein zweites Engine-Backend konkret geplant? Der Core ist heute nicht Unity-frei. [Code] Soll das neue UI-Datenmodell (IDs, Datensätze, Rollen) streng Unity-frei sein? Ich empfehle ja, weil es billig ist. Soll außerdem `UIManager.Roles` aus dem Unity-abhängigen `UIManager` herausgezogen werden?
3. **Warum wurden Inspector-Verknüpfungen bisher vermieden?** Ist der Grund nur „Szenenobjekte sind aus `Resources` nicht erreichbar“, oder gibt es weitere (Build-Größe, Module als unabhängige Pakete, Merge-Konflikte)? Ist Addressables ein Thema?
4. **Zielgeräte und Bedienung:** Tablet mit Touch, Desktop mit Maus, Controller? Davon hängen Hover (Szenario 1) und Verschieben (Szenario 6) ab.
5. **Rollen:** Ist die heutige Rangfolge (`EXPERT ⊃ … ⊃ VIEWER`) gewollt oder historisch? Sollen Nutzer mehrere Rollen gleichzeitig haben? Wer legt neue Rollen fest?
6. **Mehrere Menüs gleichzeitig:** Soll ein Floating-Menu neben dem Hauptmenü offen sein können?
7. **Arbeitsweise der Designerin:** Arbeitet sie direkt in Unity (Prefab-Editing, Nested Prefabs, Prefab Variants)? Wie werden Prefab-Änderungen in Git zusammengeführt? Prefab-YAML erzeugt schwer lösbare Merge-Konflikte. [Annahme]
8. **Bestehende Optik:** Muss sie für alte Menüs exakt erhalten bleiben (die Default-Palette entspricht den heutigen Prefabs), oder darf sich das Aussehen in Stufe 1 schon ändern?
9. **Wer pflegt den Vertrag** (ID-Konstanten, Feldnamen, kurze Doku pro Menü)?
10. **Gehört der SceneBrowser zu VISTA?** Ist er als Pilot akzeptiert, und welche Daten liefert `SceneStorageModule` tatsächlich? Im Mockup sind sie noch Platzhalter. [Code: `SceneBrowserModule.cs`, `GatherSceneList`]

### Was gelesen und was vermutet ist (Übersicht)

- **Aus dem Code gelesen:**
  - Aufbau von `MenuTree`/`MenuItem`
  - die beiden Switches und die 11 Prefabs im `MenuCreatorModule`
  - Stil- und Layout-Werte im Code und Kinder, die das Eltern-Layout ändern
  - die `Find`/`GetChild`-Stellen
  - das zweite UI-System mit Adapter-Muster
  - Rollenlogik (Liste vs. Rangfolge, doppelte Namensliste)
  - nur ein Menü gleichzeitig
  - Input-Aufteilung UI vs. Rest
  - Unity-Abhängigkeiten im Core
  - automatische Modul-Erkennung
  - Settings-Menü per Reflection
  - fehlender Wertebereich bei Parametern
  - nicht umgesetztes `ONHOVER`
- **Vermutet:**
  - Fehlerwirkungen (Bug bei Updates, mehrdeutige Ressourcen, Rangfolge-Falle bei neuen Rollen): abgeleitet, nicht getestet
  - Netzwerkbezug von `ParameterType`
  - Rechteck-Annahme im Farbwähler
  - Nested Prefabs als Lösung für Karten
  - Touch als Zielgerät
  - Merge-Probleme mit Prefabs
  - alle Aufwandszahlen
