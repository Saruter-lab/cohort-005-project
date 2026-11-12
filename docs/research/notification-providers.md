# Research: Notification-Provider für In-App-Notifications in Cadence

Ticket: [#3](https://github.com/Saruter-lab/cohort-005-project/issues/3), Map: [#2](https://github.com/Saruter-lab/cohort-005-project/issues/2). Stand: 2026-11-12.

Dieses Dokument sammelt Fakten, es trifft keine Wahl. Die Entscheidung fällt in [#4](https://github.com/Saruter-lab/cohort-005-project/issues/4).

Quellen: offizielle Docs und Pricing-Seiten der Anbieter, npm-Registry (`npm view <pkg> peerDependencies`) und GitHub-Repos der Anbieter. Keine Accounts angelegt, nichts ausprobiert. Wo eine Quelle eine Frage nicht beantwortet, steht das ausdrücklich da.

## Ausgangslage im Repo

Aus `package.json`:

- `react` / `react-dom`: `^19.2.7`
- `react-router`, `@react-router/node`, `@react-router/serve`, `@react-router/dev`: **`8.3.0`**

Wichtig: Das Repo läuft nicht auf React Router 7, sondern auf **React Router 8.3.0**. Keine der untersuchten Inbox-Bibliotheken hängt von `react-router` ab (weder als Dependency noch als Peer), die Router-Version ist also für die Paketauflösung irrelevant. Relevant ist nur die React-Peer-Range und ob die Komponenten SSR vertragen (React Router rendert serverseitig).

## Vergleichstabelle

| | Knock | Novu | Courier | MagicBell | SuprSend |
|---|---|---|---|---|---|
| Free Tier | 10.000 Messages/Monat, Profile unbegrenzt, Environments unbegrenzt | 10.000 Workflow-Runs/Monat, Subscriber unbegrenzt, 5 Workflows, 2 Environments, 24 h Activity-Retention | 10.000 Sends/Monat | 1.000 Deliveries/Monat | 10.000 Notifications/Monat |
| Danach | Starter 250 $/Monat (50.000 Messages, dann 0,005 $/Message) | Pro ab 30 $/Monat (30.000 Runs), Team ab 250 $; 1,20 $ pro 1.000 Runs extra | Business: 0,005 $ pro Send, ab dem ersten | Startup 249 $/Monat (50.000 Deliveries, dann 0,0025 $) | Essentials 100 $/Monat (50.000, dann 2 $/1.000), Business 250 $/Monat |
| React-Paket | `@knocklabs/react` 0.15.1 | `@novu/react` 3.19.2 | `@trycourier/courier-react` 9.3.5 | `@magicbell/react` 1.14.0 | `@suprsend/react` 1.5.0 |
| React-Peer-Range | `^17 \|\| ^18 \|\| ^19` | `^18 \|\| ^19` | `>=18.0.0` | keine Peer-Deps deklariert | `>= 16.8.0` |
| React 19 laut Metadaten | ja | ja | ja (offene Range) | nicht deklariert | ja (offene Range) |
| Eigene Renderer | `renderItem`, CSS-Variablen `--rnf-*`, Headless-Hooks | `renderNotification`, `renderBody` u. a., `appearance`, Headless-Hooks | eigene List-Items/Header, Themes, `useCourier()` für Custom UI | `ItemComponent`, CSS-Variablen | `notificationComponent`, `theme`, Headless über `@suprsend/react-core` |
| Beliebige User-IDs | ja, max. 256 Zeichen, ohne `/` und `#` | ja, frei wählbar, stabil und eindeutig | ja (`user_id`), Format-Regeln nicht gefunden | ja (`external_id`, String) | ja (`distinct_id`) |
| Anlage beim Trigger | ja (Inline Identification) | ja ("just in time") | nicht dokumentiert gefunden | ja (Upsert) | ja (Upsert), mit Einschränkung |
| Client-Absicherung | JWT RS256 ("Enhanced Security Mode") | HMAC-SHA256 über `subscriberId` | JWT via `POST /auth/issue-token` | JWT HS256 mit Project Secret | JWT ES256 ("Enhanced Security Mode") |
| Node-SDK | `@knocklabs/node` | `@novu/api` (`@novu/node` deprecated) | `@trycourier/courier` | `magicbell-js` | `@suprsend/node-sdk` |
| Realtime | WebSocket | WebSocket (`socketUrl`) | WebSocket (`listenForUpdates()`) | persistente Verbindung (WebSocket-Lib als Dependency) | WebSocket |
| Gelesen / Archiv | seen, read, archived (+ unarchive) | read/unread, archive, snooze | read, opened, archived | read/unread, archive, delete | seen, read, unread, archived, interacted |
| Deep-Links | `action_url` im Template, `onNotificationClick` | Redirect + `routerPush` | über `data` durchgereicht, Feldname nicht dokumentiert gefunden | in den gelesenen Seiten nicht dokumentiert | Click-Handler, `action_url` nicht dokumentiert gefunden |
| Weitere Kanäle | ja | ja | ja | ja (E-Mail, Push, Web Push, SMS, Slack, Teams) | ja |
| Dev/Test | Development-Environment, Promotion nach Prod | Dev + Prod im Free Tier | Test- und Production-Environment je Workspace | Projekte (Startup: 5 inklusive) | Sandbox, Staging (Test Mode), Production |
| Self-Hosting | keins gefunden | ja, Open Core (MIT) + Enterprise | keins gefunden | keins gefunden | nur Enterprise |

## Knock

**Free Tier.** 0 $/Monat, bis 10.000 Messages/Monat, unbegrenzt viele Environments, gespeicherte Profile unbegrenzt. Kein hartes Limit: bei Überschreitung meldet sich Knock wegen einer Zahlungsmethode. Starter kostet 250 $/Monat, ab 50.000 Messages 0,005 $ pro Message. ([Pricing](https://knock.app/pricing))

**React-Komponente.** `@knocklabs/react` bringt `KnockFeedProvider`, `NotificationIconButton`, `NotificationFeedPopover` und `NotificationFeed`. Zellen lassen sich über `renderItem` komplett ersetzen, Styling über CSS-Variablen mit Präfix `--rnf-` oder durch Ersetzen des CSS (alle Klassen beginnen mit `rnf-`), Dark Mode über `colorMode`. Headless-UI über Low-Level-Primitives (`@knocklabs/react-core`, `@knocklabs/client`). ([Feed-Docs](https://docs.knock.app/in-app-ui/react/feed), [In-App-Overview](https://docs.knock.app/in-app-ui/overview))

**Kompatibilität.** Peer: `react ^17 || ^18 || ^19`. Zusätzlich optionale Peers `next` und `@tanstack/react-router` (beide `optional: true` in `peerDependenciesMeta`), React Router wird nicht verlangt. (npm-Registry, `@knocklabs/react@0.15.1`)

**User-Identifizierung.** IDs bis 256 Zeichen, ohne `/` und `#`, nach dem Setzen nicht änderbar; Knock empfiehlt den Primärschlüssel. ([Users](https://docs.knock.app/concepts/users)) Inline Identification beim Trigger legt fehlende User an: "This enables lazy recipient creation within Knock". Recipients müssen Objekte sein (`{ id: "user-123" }`), nicht nackte Strings. ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients))

**Client-Absicherung.** Ohne Enhanced Security Mode gilt: "a request with your public key can act as any user". Mit Enhanced Security signiert das Backend ein JWT per RS256 mit dem privaten Signing Key, `sub` = User-ID. Standardmäßig aus, für Production empfohlen. ([Security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication))

**Server-SDK.** `@knocklabs/node` (1.41.1), Trigger über `knock.workflows.trigger(key, { recipients, data })`. ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients))

**Realtime.** WebSocket, "no infrastructure setup required". Optional automatisches Trennen bei inaktiven Tabs (`auto_manage_socket_connection`). ([In-App-Overview](https://docs.knock.app/in-app-ui/overview), [Feed-Docs](https://docs.knock.app/in-app-ui/react/feed))

**Status und Deep-Links.** seen, read, archived; archivierte Nachrichten sind standardmäßig ausgeblendet und lassen sich wiederherstellen (`markAsArchived`, REST `PUT/DELETE /v1/messages/{id}/archived`). `action_url` im Template erzeugt automatisch einen Redirect-Handler, überschreibbar mit `onNotificationClick`. ([FeedClient](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client), [Messages API](https://docs.knock.app/api-reference/messages), [Feed-Docs](https://docs.knock.app/in-app-ui/react/feed))

**Dev/Test.** Start mit Production, zuschaltbar Development und weitere Environments; Inhalte werden nach oben promotet, jedes Environment hat eigene Keys und isolierte Daten. Ein eigener Mock-/Sandbox-Modus ist in der Environment-Doku nicht beschrieben. ([Environments](https://docs.knock.app/concepts/environments))

**Self-Hosting.** Nicht erwähnt. Die JS-SDKs sind MIT-lizenziert ([knocklabs/javascript](https://github.com/knocklabs/javascript)), der Dienst selbst nicht.

## Novu

**Free Tier.** 0 $/Monat, 10.000 Workflow-Runs/Monat, Subscriber unbegrenzt, 5 Workflows, 2 Environments (Dev + Prod), 3 Team-Mitglieder, 24 h Activity-Feed-Retention. Pro ab 30 $/Monat (30.000 Runs, 20 Workflows), Team ab 250 $/Monat; Overage 1,20 $ pro 1.000 Runs. Was beim Überschreiten im Free Tier passiert, steht nicht auf der Seite. ([Pricing](https://novu.co/pricing))

Achtung Abrechnungseinheit: Novu zählt Workflow-Runs, nicht Nachrichten. Die Zahlen sind daher nicht 1:1 mit Knock/Courier vergleichbar.

**React-Komponente.** `@novu/react` mit `Inbox`, `Bell` (`renderBell`), `Notifications`, `InboxContent`. Render-Props `renderNotification`, `renderSubject`, `renderBody`, `renderAvatar`, `renderDefaultActions`; Styling über `appearance` (`variables`, `elements`, `icons`, `baseTheme`). Headless-Hooks vorhanden. `routerPush` zum Einhängen der Router-Navigation. ([React SDK](https://docs.novu.co/platform/sdks/react), [Inbox-Overview](https://docs.novu.co/platform/inbox/overview))

**Kompatibilität.** Peer: `react ^18.0.0 || ^19.0.0 || ^19.0.0-0`. (npm-Registry, `@novu/react@3.19.2`)

**User-Identifizierung.** `subscriberId` frei wählbar, muss im Environment eindeutig und stabil sein; Novu empfiehlt die interne User-ID. Beim Trigger wird ein fehlender Subscriber "just in time" aus dem `to`-Objekt angelegt, bestehende werden aktualisiert. Zusätzlich legt die Inbox neue Subscriber beim ersten Session-Start an. ([Subscribers](https://docs.novu.co/platform/concepts/subscribers), [Inbox-Overview](https://docs.novu.co/platform/inbox/overview))

**Client-Absicherung.** HMAC: `subscriberHash = HMAC-SHA256(NOVU_SECRET_KEY, subscriberId)` als Hex, serverseitig berechnet, im Dashboard pro Environment einzuschalten. Ist HMAC aktiv und fehlt der Hash, lädt die Inbox nicht. ([Prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production))

**Server-SDK.** `@novu/api` (3.19.1). `@novu/node` ist laut npm seit 20.03.2025 deprecated. (npm-Registry)

**Realtime.** Inbox nimmt `socketUrl`/`socketOptions`, Status wird "in real-time" synchronisiert. ([React SDK](https://docs.novu.co/platform/sdks/react))

**Status und Deep-Links.** read/unread, archive/unarchive, snooze/unsnooze; Klick-Handler `onNotificationClick`, `onPrimaryActionClick`, `onSecondaryActionClick`. ([React SDK](https://docs.novu.co/platform/sdks/react))

**Self-Hosting.** Ja. Open Core: Kern MIT, Ordner `enterprise/` unter kommerzieller Lizenz ([README](https://github.com/novuhq/novu#license)). Self-Hosting braucht API, Worker, WS, Dashboard, MongoDB, Redis und Object Storage; einige Cloud-Features fehlen self-hosted, Social Login wird nicht unterstützt. ([Self-Hosting-Overview](https://docs.novu.co/community/self-hosting-novu/overview)) Für Cadence (eine SQLite-Datei, siehe ADR 0006) wäre das ein erheblicher Infrastruktur-Sprung.

## Courier

**Free Tier.** Developer-Plan bis 10.000 Sends/Monat. Darüber Business mit 0,005 $ pro Send, und zwar inklusive der ersten 10.000 (50.000 Sends = 250 $). Enterprise mit individuellen Preisen. ([Pricing](https://www.courier.com/pricing))

**React-Komponente.** `@trycourier/courier-react` mit `CourierInbox`, `CourierInboxPopupMenu`, `CourierToast`, `CourierPreferences`, Hook `useCourier()` für Custom UIs. Themes, eigene Header und List-Items, eigene Empty/Loading/Error-States. ([React-SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web), [Inbox](https://www.courier.com/docs/in-app/inbox))

**Kompatibilität.** Peer: `react >=18.0.0`. Für React 17 gibt es ein separates Paket. Inbox und Toast "render client-side only", in Next.js braucht es `'use client'`. Für React Router heißt das: Komponente nur clientseitig rendern. (npm-Registry, [React-SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web))

**User-Identifizierung.** Senden an `to.user_id`; die ID muss mit der `userId` beim `signIn` übereinstimmen, sonst wird nicht zugestellt. Ob der User vorher existieren muss, sagt die gelesene Seite nicht. ([Send to the inbox](https://www.courier.com/docs/in-app/send-to-the-inbox))

**Client-Absicherung.** Backend ruft `POST /auth/issue-token` mit API-Key und Scope-String auf (`user_id:<id> inbox:read:messages inbox:write:events`). Ohne `expires_in` läuft das Token nie ab; die SDKs refreshen nicht selbst. ([Authentication](https://www.courier.com/docs/platform/inbox/authentication))

**Server-SDK.** `@trycourier/courier` (10.0.0), `client.send.message({ message: { to, template, data, routing: { channels: ["inbox"] } } })`. Ein veröffentlichtes Template mit In-App-Kanal ist Voraussetzung. ([Send to the inbox](https://www.courier.com/docs/in-app/send-to-the-inbox))

**Realtime.** `inbox.listenForUpdates()` öffnet die WebSocket-Verbindung; Zustandsänderungen synchronisieren über Geräte. ([React-SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web), [Inbox-Overview](https://www.courier.com/docs/platform/inbox/inbox-overview))

**Status und Deep-Links.** `read`, `opened`, `archived`; Unread- und Archived-Views. Zusätzliche Felder in `data` werden "untouched" durchgereicht, "useful for deep linking". Ein dediziertes Action-URL-Feld fand sich auf den gelesenen Seiten nicht. ([Inbox](https://www.courier.com/docs/in-app/inbox), [Send to the inbox](https://www.courier.com/docs/in-app/send-to-the-inbox))

**Dev/Test.** Jeder Workspace hat Production und Test, bis zu 20 Environments; Key-Typen Published (`pk_`) und Draft (`dk_`). ([Environments and API keys](https://courier.com/docs/platform/workspaces/environments-api-keys)) Ein Mock-Key, der den Lebenszyklus ohne Provider simuliert, wird laut Suchergebnis-Auszug der Courier-Docs nur im Enterprise-Plan angeboten; auf der direkt abgerufenen Seite war er nicht zu finden, also unbestätigt. Sends aus allen Environments zählen für die Abrechnung.

**Self-Hosting.** Keins gefunden.

## MagicBell

**Free Tier.** Builder: 0 $/Monat, 1.000 Deliveries/Monat; kein MAU-Limit genannt. Startup 249 $/Monat mit 50.000 Deliveries und 5 Projekten, dann 0,0025 $ pro Delivery und 99 $ pro zusätzlichem Projekt. ([Pricing](https://www.magicbell.com/pricing))

**React-Komponente.** `@magicbell/react` mit `ContextProvider` (`token`), `FloatingInbox`, `Inbox`, `InlineNotification`, `ToastContainer`, `UserPreferences`. Eigene Renderer über `ItemComponent`, `ButtonComponent`, `InboxComponent`; Styling über CSS-Klassen und Variablen (`--magicbell-*`). Headless-Hooks auf der Seite nicht beschrieben. ([React-Docs](https://www.magicbell.com/docs/libraries/magicbell-react))

**Kompatibilität.** `@magicbell/react@1.14.0` deklariert keine `peerDependencies`, die React-Version ist also aus den Metadaten nicht ablesbar. Das ältere `@magicbell/magicbell-react` (11.5.1) hat Peer `react >= 18.3.1`. (npm-Registry)

**User-Identifizierung.** Über `external_id` oder E-Mail, empfohlen ist `external_id`; nicht änderbar, Integer müssen als String übergeben werden. "If a user doesn't exist, a new user will be created." ([User](https://www.magicbell.com/docs/primitive/user))

**Client-Absicherung.** User-JWT, HS256 mit dem Project Secret Key, Claims `user_external_id` und/oder `user_email` plus `api_key`. ([User JWT](https://www.magicbell.com/docs/api/authentication/user))

**Server-SDK.** `magicbell-js`, `ProjectClient` aus `magicbell-js/project-client`, Versand über `client.broadcasts.createBroadcast(...)` mit `recipients`. ([magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js))

**Realtime.** "MagicBell maintains a persistent connection to deliver notifications in real-time"; das React-Paket hat `reconnecting-websocket` als Dependency. ([In-App](https://www.magicbell.com/docs/channels/in_app))

**Status und Deep-Links.** Lesen, als gelesen markieren, archivieren, löschen. Ein Action-URL-Feld ist auf den gelesenen Seiten nicht dokumentiert. ([In-App](https://www.magicbell.com/docs/channels/in_app))

**Weitere Kanäle.** E-Mail (SES, Mailgun, SMTP, SendGrid), Mobile Push (APNs, FCM, Expo), Web Push, SMS (Twilio), Slack, Teams. ([Docs-Übersicht](https://www.magicbell.com/docs))

**Self-Hosting.** Keins gefunden.

## SuprSend

**Free Tier.** 0 $/Monat, 10.000 Notifications/Monat, Team unbegrenzt; Überschreiten stoppt den Versand nicht, SuprSend meldet sich. Essentials 100 $/Monat (50.000, dann 2 $ pro 1.000), Business 250 $/Monat (50.000, dann 5 $ pro 1.000). ([Pricing](https://www.suprsend.com/pricing))

**React-Komponente.** Aktuelles Paket `@suprsend/react`; `@suprsend/react-inbox` ist end-of-support und auf npm deprecated. `SuprSendProvider` + `Inbox` (Popover) bzw. `NotificationFeed`; Anpassung über `notificationComponent`, `bellComponent`, `badgeComponent`, `theme`/`themeType`, Click-Handler. Headless über `@suprsend/react-core`. ([Inbox React](https://docs.suprsend.com/docs/inbox-react), [Popover feed](https://docs.suprsend.com/docs/react-in-app-feed))

**Kompatibilität.** Peer: `react >= 16.8.0`. Dependencies enthalten u. a. `react-markdown@6.0.3`, `remark-gfm@1.0.0`, `@emotion/react`, `react-popper`; ob die gepinnten älteren Markdown-Pakete unter React 19 sauber laufen, ist aus den Metadaten nicht ableitbar. (npm-Registry)

**User-Identifizierung.** `distinct_id` aus dem eigenen System, unveränderlich, pro Workspace getrennt. Beim Trigger "SuprSend upserts the user before sending", es sei denn `$skip_create: true`. Einschränkung: Nur `distinct_id` ohne Kanäle lässt den Trigger scheitern; ob für den Inbox-Kanal eine eigene Channel-Identity nötig ist, geht aus der Seite nicht hervor. ([Users](https://docs.suprsend.com/docs/users))

**Client-Absicherung.** Enhanced Security Mode: Backend signiert JWT mit ES256, Claims `entity_type: "subscriber"`, `entity_id`, `exp`, `iat`, optional `scope.tenant_id`. Client ruft `identify(id, userToken)`, optional `refreshUserToken`. ([Client authentication](https://docs.suprsend.com/docs/client-authentication))

**Server-SDK.** `@suprsend/node-sdk` (1.15.1). (npm-Registry)

**Realtime.** WebSocket, `initializeSocketConnection()`, Reachability-Status ab `@suprsend/react` 1.3.0. ([JS In-App Feed](https://docs.suprsend.com/docs/js-inapp-feed), [Popover feed](https://docs.suprsend.com/docs/react-in-app-feed))

**Status und Deep-Links.** `markAsSeen`, `markAsRead`, `markAsUnread`, `markAsArchived`, `markAsInteracted`, `markAllAsRead`; mehrere "Stores" (Tabs). Action-URL-Feld nicht dokumentiert gefunden, Klicks laufen über `notificationClickHandler` bzw. Action-Button-Handler. ([JS In-App Feed](https://docs.suprsend.com/docs/js-inapp-feed), [Popover feed](https://docs.suprsend.com/docs/react-in-app-feed))

**Dev/Test.** Drei Workspaces: Sandbox (Demo, 30 Tage), Staging mit Test Mode (sendet nur an Tester), Production; Promotion per CLI (Beta). ([Workspace-Doku via Suche](https://docs.suprsend.com/docs/guides/modelling-customers-workspace-vs-tenant))

**Self-Hosting.** Nur im Enterprise-Plan ([Pricing](https://www.suprsend.com/pricing), [Self-hosted Docs](https://docs.suprsend.com/docs/self-hosted/cli-profiles)).

## Querschnittsfragen für Cadence

**DevUI-User-Switching.** Alle fünf akzeptieren eigene, stabile User-IDs. Bei Knock, Novu, MagicBell und SuprSend sind IDs nach dem Anlegen unveränderlich, die Cadence-User-ID (Primärschlüssel) passt dazu. Beim Wechsel im DevUI muss der Client neu authentifiziert werden: Knock, Courier, MagicBell, SuprSend brauchen dafür ein neues serverseitig signiertes JWT, Novu einen neuen HMAC-Hash. Alle Verfahren lassen sich in einer React-Router-Loader-Funktion erzeugen.

**SSR.** Courier dokumentiert ausdrücklich client-only Rendering. Für die anderen ist SSR-Verhalten in den gelesenen Seiten nicht beschrieben; Inbox-Komponenten werden in der Praxis meist nur clientseitig gerendert. Muss im Prototyp geprüft werden.

**Lokale Entwicklung ohne echte Keys.** Kein Anbieter bietet laut gelesener Doku einen lokalen Emulator. Alle haben getrennte Dev-/Test-Environments mit eigenen Keys. Für Tests ohne Netz bleibt ein eigener Fake hinter einem Adapter, unabhängig vom Anbieter. Novu ist der einzige, der sich (aufwendig) komplett lokal betreiben lässt.

## Ausschlusskriterien und Auffälligkeiten (Fakten, keine Wertung)

- **MagicBell**: Free Tier mit 1.000 Deliveries/Monat deutlich kleiner als bei den anderen (10.000). React-Paket ohne deklarierte Peer-Range.
- **Courier**: Business-Preis zählt ab dem ersten Send; React-Komponenten nur client-seitig; Mock-Keys (falls bestätigt) nur Enterprise.
- **Novu**: abgerechnet in Workflow-Runs, Free Tier auf 5 Workflows begrenzt (relevant, wenn der Event-Katalog mehr als 5 Notification-Arten hat), Activity-Retention 24 h.
- **SuprSend**: altes Inbox-Paket deprecated, neues Paket mit älteren gepinnten Markdown-Dependencies.
- **Knock**: Enhanced Security standardmäßig aus, ohne sie kann der Public Key jeden User spielen. Gilt analog für Novu-HMAC und SuprSend-Enhanced-Security.

## Offene Unsicherheiten

- Courier: Wird ein unbekannter `user_id` beim Send automatisch als Profil angelegt? Gibt es ein dediziertes Action-URL-Feld? Mock-Key-Plan unbestätigt.
- MagicBell: React-19-Kompatibilität von `@magicbell/react` nicht deklariert; Action-URL-Feld nicht gefunden; Free-Tier-Projektanzahl nicht angegeben.
- SuprSend: Braucht der Inbox-Kanal eine Channel-Identity am User, damit der Inline-Trigger nicht scheitert? Läuft `@suprsend/react` mit seinen Dependencies unter React 19 ohne Warnungen?
- Alle: SSR-Verhalten unter React Router 8 (Repo nutzt 8.3.0, nicht 7) ist nicht dokumentiert und sollte in einem Prototyp geprüft werden.
- Nicht untersucht: weitere Anbieter wie Engagespot oder Customer.io In-App.
