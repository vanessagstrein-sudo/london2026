/* LONDON 2026 – Offline-Unterstützung
   Speichert die App, die Fotos und die Schriften auf dem Handy,
   damit alles auch ohne Internet (Flugzeug, U-Bahn) funktioniert.
   Bei Internetverbindung wird immer die neueste Version von index.html geladen. */
const CACHE = "london2026-v2";
const W = (p, n, w = 960) => `https://upload.wikimedia.org/wikipedia/commons/thumb/${p}/${n}/${w}px-${n}`;
const PRECACHE = [
  "./",
  "./index.html",
  W("6/67", "London_Skyline_%28125508655%29.jpeg", 1280),
  W("f/ff", "Brentford_Community_Stadium_2020.jpg"),
  W("4/4a", "Kensington_Olympia_exhibition_centre_from_station.jpg"),
  W("c/c8", "Wiki_takes_Nordtiroler_Oberland_20150604_L%C3%A4ngenfeld_6275.jpg")
];

self.addEventListener("install", (event) => {
  event.waitUntil(
    caches.open(CACHE).then((cache) =>
      Promise.all(
        PRECACHE.map((url) => {
          const external = url.startsWith("http");
          return fetch(new Request(url, external ? { mode: "no-cors" } : {}))
            .then((res) => cache.put(url, res))
            .catch(() => {});
        })
      )
    )
  );
  self.skipWaiting();
});

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys()
      .then((keys) => Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k))))
      .then(() => self.clients.claim())
  );
});

self.addEventListener("fetch", (event) => {
  const req = event.request;
  if (req.method !== "GET") return;

  // Seite selbst: zuerst Netz (für Updates), sonst gespeicherte Version
  if (req.mode === "navigate") {
    event.respondWith(
      fetch(req)
        .then((res) => {
          const copy = res.clone();
          caches.open(CACHE).then((c) => c.put("./index.html", copy));
          return res;
        })
        .catch(() => caches.match("./index.html").then((r) => r || caches.match("./")))
    );
    return;
  }

  // Fotos, Schriften usw.: zuerst Speicher, sonst Netz (und dann speichern)
  event.respondWith(
    caches.match(req).then((cached) => {
      if (cached) return cached;
      return fetch(req)
        .then((res) => {
          const copy = res.clone();
          caches.open(CACHE).then((c) => c.put(req, copy));
          return res;
        })
        .catch(() => cached);
    })
  );
});
