# Tiket Engineer — Info Data

Internal web app for PT. Info Solusindo Data Utama for logging and tracking engineer cases across all teams.

- [`PLAN.md`](PLAN.md): product and technical plan
- [`index.html`](index.html): clickable UI mockup, live at https://muhghifari.github.io/infodata-ticketing/ (GitHub Pages deploys from the root of the `main` branch)

## UI mockup

`index.html` is a clickable prototype. It's one file: open it in any browser. There is no backend; the sample data is generated in the browser and resets on reload.

Screens: Dashboard, Papan Tiket (kanban, drag cards between columns), Tiket Saya, Daftar Tiket (sortable, CSV export), ticket detail drawer (fields, SLA, history, work notes), Buat Tiket form, and Admin (SLA policy, teams, users).

Mockup simplifications: SLA uses calendar hours (the real app will use business hours and holidays); Admin forms for teams and users are placeholders.
