# HEIC to JPG

A simple, privacy-friendly HEIC and HEIF to JPG converter built with Vue.

The conversion runs entirely in the browser. Images are not uploaded to a
server.

Live app: [heic.chleb.app](https://heic.chleb.app)

## Features

- Converts HEIC and HEIF images to JPG
- Adjustable JPG quality
- Image preview and output size summary
- Local, browser-only processing
- Drag-and-drop file selection

## Development

```bash
npm install
npm run dev
```

Create a production build with:

```bash
npm run build
```

## Docker

```bash
docker compose up -d --build
```

The production container joins the external Docker network `chleb-apps` and is available to a reverse proxy at `http://heic-to-jpg:80`.
