# Mystery Countdown Svelte

A Svelte + Vite birthday countdown inspired by the supplied Lovable preview URLs.

## Features

- Mystery countdown to **15.09.2026**
- `/` landing page with live days/hours/minutes/seconds
- `/geburtstag` birthday reveal page
- Built-in Web Audio "Happy Birthday" melody (starts from a user click, as browsers require)
- Responsive glassmorphism birthday design with animated confetti
- Docker + Docker Compose deployment through nginx

## Run locally

```bash
npm install
npm run dev
```

Open:

- http://localhost:5173/
- http://localhost:5173/geburtstag

## Build

```bash
npm run build
npm run preview
```

## Deploy with Docker Compose

```bash
docker compose up -d --build
```

Open:

- http://localhost:8080/
- http://localhost:8080/geburtstag

If port 8080 is already used:

```bash
HOST_PORT=8090 docker compose up -d --build
```

Stop it:

```bash
docker compose down
```
