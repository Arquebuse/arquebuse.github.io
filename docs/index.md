# Home

Arquebuse is an open-source notification system tailored specifically for enterprise environments, providing secure and efficient delivery of targeted notifications to Windows desktop users.

Designed with simplicity at its core, Arquebuse integrates seamlessly with existing infrastructure, allowing administrators to easily select recipients, send critical application outage alerts, and request user acknowledgments directly via the native Windows notification center. 

It respects users' "Do Not Disturb" settings, ensuring alerts are delivered respectfully without disruption. Arquebuse simplifies enterprise communication, empowering organizations to effectively manage notifications at scale.

## Modules

Arquebuse is composed of loosely coupled modules:

- [**Desktop‑App**](https://github.com/Arquebuse/desktop-app) - A Windows service that registers with WNS, receives and displays notifications.
- [**Controller‑Core**](https://github.com/Arquebuse/controller-core) - A .NET 8 Minimal API managing client registrations, AD queries, and sending WNS notifications (uses SQLite).
- [**Controller‑CLI**](https://github.com/Arquebuse/controller-cli) - A PowerShell module to manage subscriptions and send notifications via Controller‑Core.

## Goals

- Deliver targeted Windows notifications at scale.
- Integrate with Active Directory for precise targeting.
- Maintain simplicity, security, and enterprise readiness.

## Technologies

- **Desktop‑App**: .NET 8 Windows Service, WNS integration, JSON logging
- **Controller‑Core**: .NET 8 Minimal API, Kestrel (HTTPS), System.DirectoryServices (LDAP), SQLite
- **Controller‑CLI**: PowerShell 7+ module, REST API over HTTPS

## Project status

We are in the early stage of the project. Nothing is ready to use, but stay tuned, we move fast.

## Vibe coding

This project and its documentation were created and maintained using AI tools like ChatGPT and GitHub Copilot. Feel free to mock the code logic, style, or anything else, no one will take offense 😉