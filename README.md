<p align="center">
  <a href="https://agentsdotne.uk"><img src="docs/assets/banner.svg" alt="AGENTSdotNE Cohort base app" width="100%"></a>
</p>

<p align="center">
  <a href="https://agentsdotne.uk"><img src="https://img.shields.io/badge/AGENTSdotNE-cohort-C5E63A?style=flat-square&labelColor=121212" alt="AGENTSdotNE cohort"></a>
  <img src="https://img.shields.io/badge/Django-5.2-C5E63A?style=flat-square&labelColor=121212&logo=django&logoColor=C5E63A" alt="Django 5.2">
  <img src="https://img.shields.io/badge/React-19-C5E63A?style=flat-square&labelColor=121212&logo=react&logoColor=C5E63A" alt="React 19">
  <img src="https://img.shields.io/badge/TypeScript-5.9-C5E63A?style=flat-square&labelColor=121212&logo=typescript&logoColor=C5E63A" alt="TypeScript 5.9">
  <img src="https://img.shields.io/badge/Tailwind-4-C5E63A?style=flat-square&labelColor=121212&logo=tailwindcss&logoColor=C5E63A" alt="Tailwind 4">
  <img src="https://img.shields.io/badge/shadcn%2Fui-components-C5E63A?style=flat-square&labelColor=121212&logo=shadcnui&logoColor=C5E63A" alt="shadcn/ui components">
  <img src="https://img.shields.io/badge/Storybook-10-C5E63A?style=flat-square&labelColor=121212&logo=storybook&logoColor=C5E63A" alt="Storybook 10">
  <img src="https://img.shields.io/badge/Postgres-17-C5E63A?style=flat-square&labelColor=121212&logo=postgresql&logoColor=C5E63A" alt="Postgres 17">
</p>

**Staging:** [cohort-1c-production.up.railway.app](https://cohort-1c-production.up.railway.app) · [what works differently there](docs/staging.md)

The base application gives your team a working starting point for the six-week project. It already includes
account creation, sign-in, password recovery and account settings. It also provides the navigation around the
application, including a sidebar, mobile menu and user menu. Every screen is built from shadcn/ui components,
so new screens can reuse the same building blocks and look consistent from day one.

The dashboard is deliberately empty. Your team will decide what belongs there after understanding the current
cohort's problem and agreeing a manageable user journey.

## What you get

| | |
| --- | --- |
| Backend | Django 5.2, Django REST Framework, Postgres |
| Frontend | React 19, Vite, TypeScript, Tailwind 4, shadcn/ui |
| Components | shadcn/ui in `src/components/ui`, and Storybook for viewing components on their own |
| Auth | Email and password, session cookies, change password. Email confirmation and password reset are built but switched off |
| Uploads | Profile pictures, stored in `backend/media/` in development |
| Email | Templated HTML email, off until `EMAIL_ENABLED=true` and an SMTP server are set |
| Data | TanStack Query, a typed `apiRequest` wrapper, react-hook-form and zod |
| Checks | ruff, Django tests, ESLint, Prettier and TypeScript, run locally and in GitHub Actions |

## Getting started

Each team works in its own fork of this repository, which your facilitator sets up. Clone your team's fork.

### The quick way: everything in Docker

With [Docker](https://docs.docker.com/get-docker/) running, one command starts the database, Django and
React together:

```bash
make dev              # or: docker compose up
```

The first start installs every package, so give it a few minutes. Both servers reload when you save a file,
and `Ctrl+C` stops them all.

### Running the servers on your machine

You also need this for `make test` and for your editor's type checking. Install
[uv](https://docs.astral.sh/uv/) and Node 24. With [nvm](https://github.com/nvm-sh/nvm),
`cd frontend && nvm install` picks the right Node from `frontend/.nvmrc`.

```bash
make install          # backend packages with uv, frontend packages with npm
make db               # start only Postgres in Docker, on port 5442
make migrate          # create the tables
```

Then run the two servers, each in its own terminal:

```bash
make backend          # Django on http://127.0.0.1:8000
make frontend         # React on http://localhost:5173
```

### Your first account

Open http://localhost:5173 and create an account. Email is switched off, locally and on staging, so you are
signed in straight away.

Every setting has a working default. To change one, copy `backend/.env.example` to `backend/.env`.

## Everyday commands

```bash
make test             # every check CI runs: ruff, migrations, Django tests, ESLint, Prettier, TypeScript, build
make format           # fix formatting in the backend and the frontend
make seed             # fictional accounts, including a staff one for /admin/ (needs make dev)
make superuser        # create an admin account for http://localhost:8000/admin/ (needs make dev)
make storybook        # browse components on their own at http://localhost:6006 (needs make dev)
```

## Where things are

```
backend/
  config/       settings and the top-level URLs
  accounts/     the user model, sign-in endpoints, email sending
frontend/src/
  app/          router, providers, the signed-in shell and its navigation
  auth/         sign in, sign up, confirm email, reset password, useSession
  account/      the account settings dialog: picture, name and password
  components/   shared components, plus ui/ from shadcn
  lib/          apiRequest, useApiForm, the query client, cn
  pages/        standalone pages, like the dashboard
docs/           how things work, and how to add to them
example_inbound.csv   fictional inbound items for the brief, as they arrive
```

## UI components

The building blocks live in `frontend/src/components/ui`: alert, avatar, button, card, dialog, dropdown
menu, input, label, separator, sheet, skeleton and sonner (toasts). They come from
[shadcn/ui](https://ui.shadcn.com), which copies each component's source into the repo instead of installing
a package. That means you own the code: read it, and change it if a design needs it.

The design is yours. The sidebar, header and neutral palette are a starting point, not a house style: swap
the sidebar for a top bar, change the layout, colours and type, or restyle any component, to whatever fits
your users.

- **Check before you build.** If `components/ui` or the [shadcn catalogue](https://ui.shadcn.com/docs/components)
  already has it, use that rather than writing your own.
- **Add one** with `cd frontend && npx shadcn@latest add <name>`. If the new file imports from `"cn"`, see
  [troubleshooting](docs/troubleshooting.md#a-shadcn-component-imports-from-cn).
- **Build on them.** App-level pieces such as `FormField` and `UserAvatar` sit in `src/components` and are
  made from the `ui` ones.
- **See them on their own** with `make storybook`. Add a `*.stories.tsx` file next to a component to give it
  a page; `button.stories.tsx` is the example to copy.
- **Restyle them all at once** by changing the colour variables, as described in [Branding it](#branding-it).

## Guides

- **[Getting started](docs/getting-started.md).** The whole app explained from scratch: setup, a tour, how it fits together.
- **[Adding a feature](docs/adding-a-feature.md).** Start here. It builds a small to-do list, step by step.
- **[Sharing data between users](docs/sharing-data.md).** Shared records, roles and two people editing at once.
- **[Staging](docs/staging.md).** Deploying to Railway, and what works differently there.
- **[How sign-in works](docs/how-auth-works.md).** Sessions, CSRF, the Vite proxy, and every auth endpoint.
- **[Troubleshooting](docs/troubleshooting.md).** Fixes for the errors people actually hit.
- **[Working on this project](CONTRIBUTING.md).** Branches, pull requests and reviews.
- **[AGENTS.md](AGENTS.md).** The rules AI coding assistants should follow here. `CLAUDE.md` points to it.

## Adding to it

**A feature with data.** Follow [docs/adding-a-feature.md](docs/adding-a-feature.md).

**A page.** Create it in its feature folder (or `src/pages` if it stands alone), add a `<Route>` in
`src/app/router.tsx`, and add a link in `src/app/layout/app-nav.tsx`. The sidebar and the mobile menu both
read that list.

**An email.** Call `send_templated_email(to, subject, template, context)` from
`backend/accounts/emails.py`. Templates live in `backend/accounts/templates/emails/` and extend `base.html`.

## Branding it

The palette is shadcn's `neutral`, set as CSS variables in `frontend/src/index.css`. Change `--primary`
and `--primary-foreground` under both `:root` and `.dark` and the whole app follows. The app name is set
in `frontend/index.html`, `APP_NAME` in `frontend/src/app/app-name.ts`, and `SITE_NAME` in the backend
settings, which email subjects use.

## What is deliberately missing

This is a starting point, not a finished product. Before anything real:

- **Rate limiting.** Sign-in, sign-up and password reset accept unlimited attempts, so the app is open to
  password guessing and to being used to send spam. Add throttling first.
- **Roles and permissions.** Every signed-in user can use everything, and each user only sees their own
  data. Staff can manage users in Django admin at `/admin/`. [Sharing data](docs/sharing-data.md) shows the
  patterns when that is not enough.
- **Production settings.** `DEBUG` is on by default, so the app runs with no setup. Turn it off and set a
  real `SECRET_KEY` on any deployment ([staging](docs/staging.md) lists the rest), and read
  `python manage.py check --deploy` before anything real.
- **Storage for uploads.** Profile pictures are saved to `backend/media/` and served by Django. That is fine
  for development and staging. In production, store them in object storage such as S3 (`django-storages`).
- **A deployment.** The `Dockerfile` builds the frontend and backend into one image that serves both,
  ready for a host such as Railway, but nothing is deployed for you and the production settings above
  still apply.
- **Live updates.** Data is fetched when a page loads. For a live view, poll with TanStack Query's
  `refetchInterval` before reaching for WebSockets.
- **Frontend tests.** The frontend has lint and type checking only. Add Vitest and Testing Library when
  you have behaviour worth pinning down.
- **AI integration.** When you add one, put the API key in `backend/.env` and call the model from
  Django. A key shipped to the browser is a key you have given away.

---

<p align="center">
  Built for the <a href="https://agentsdotne.uk">AGENTSdotNE</a> cohort programme, North East England.
</p>
