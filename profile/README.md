# Aam Digital
_Enabling social organizations digitally to transform lives._

Aam Digital is an easy-to-use case management software for the social sector that improves the effectiveness and transparency of work with beneficiaries in the field.

<div align="center"><img src="https://github.com/Aam-Digital/ndb-core/assets/1682541/2b125750-5c03-4dc7-873f-22d8278accde"  width="30%"></div>

-----
We are building an open source (GPL-3.0) cloud platform tailored to NGOs and social sector use cases.
Aam Digital is highly customizable as a "no-code" platform and fully functional offline with synchronization between devices and users whenever connectivity is available.

We are certified as a "Digital Public Good".
Aam Digital's core team is driven by a spirit of social entrepreneurship.
To make the case management solution more widely and easily available, we offer a Software-as-a-Service version of the open source system:
For more information about the software and an open demo system visit **[www.aam-digital.com](https://www.aam-digital.com)**.

[![DPG Badge](https://img.shields.io/badge/Verified-DPG-3333AB?logo=data:image/svg%2bxml;base64,PHN2ZyB3aWR0aD0iMzEiIGhlaWdodD0iMzMiIHZpZXdCb3g9IjAgMCAzMSAzMyIgZmlsbD0ibm9uZSIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj4KPHBhdGggZD0iTTE0LjIwMDggMjEuMzY3OEwxMC4xNzM2IDE4LjAxMjRMMTEuNTIxOSAxNi40MDAzTDEzLjk5MjggMTguNDU5TDE5LjYyNjkgMTIuMjExMUwyMS4xOTA5IDEzLjYxNkwxNC4yMDA4IDIxLjM2NzhaTTI0LjYyNDEgOS4zNTEyN0wyNC44MDcxIDMuMDcyOTdMMTguODgxIDUuMTg2NjJMMTUuMzMxNCAtMi4zMzA4MmUtMDVMMTEuNzgyMSA1LjE4NjYyTDUuODU2MDEgMy4wNzI5N0w2LjAzOTA2IDkuMzUxMjdMMCAxMS4xMTc3TDMuODQ1MjEgMTYuMDg5NUwwIDIxLjA2MTJMNi4wMzkwNiAyMi44Mjc3TDUuODU2MDEgMjkuMTA2TDExLjc4MjEgMjYuOTkyM0wxNS4zMzE0IDMyLjE3OUwxOC44ODEgMjYuOTkyM0wyNC44MDcxIDI5LjEwNkwyNC42MjQxIDIyLjgyNzdMMzAuNjYzMSAyMS4wNjEyTDI2LjgxNzYgMTYuMDg5NUwzMC42NjMxIDExLjExNzdMMjQuNjI0MSA5LjM1MTI3WiIgZmlsbD0id2hpdGUiLz4KPC9zdmc+Cg==)](https://digitalpublicgoods.net/r/aam-digital)
[![User Support Center](https://img.shields.io/badge/User%20Support%20Center-available-blue)](https://chatwoot.help/hc/aam-digital/en)

-----
## Software Architecture
The core application is a progressive web app (written in TypeScript + Angular) connected to a CouchDB database server.
To provide some advanced functionality, there are some (optional) backends.
The system is deployed via docker-compose.

The repositories in this GitHub organisation reflect these different components.

Start here with the core frontend: **[ndb-core](https://github.com/Aam-Digital/ndb-core)**
