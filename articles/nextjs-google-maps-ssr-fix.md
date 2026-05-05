---
title: "Next.js App RouterでGoogle Maps APIを使う際のSSR問題を完全解決する"
emoji: "🗺️"
type: "tech"
topics: ["nextjs", "googlemaps", "typescript", "react"]
published: true
---

## 問題：Google MapsはブラウザAPIに依存する

Google Maps APIは `window` オブジェクトを使うため、SSR（サーバーサイドレンダリング）環境では動かない。Next.js App Routerでそのまま使おうとすると以下のエラーが出る。

```
Error: window is not defined
```

またはビルド時に：

```
Error: ssr: false is not allowed with next/dynamic in Server Components
```

これを正しく解決する方法を解説する。

---

## 解決策：Client Componentラッパーで `ssr: false` を使う

`ssr: false` は**Client Component**の中でしか使えない。

```
// ❌ Server Component（デフォルト）の中ではNG
import dynamic from "next/dynamic";
const Map = dynamic(() => import("./Map"), { ssr: false }); // エラー
```

```typescript
// ✅ Step 1: "use client" のラッパーを作る
// components/Map/MapClientLoader.tsx
"use client";
import dynamic from "next/dynamic";

const RealEstateMap = dynamic(
  () => import("./RealEstateMap"),
  {
    ssr: false,
    loading: () => (
      <div className="w-full h-full bg-slate-100 animate-pulse rounded-xl" />
    ),
  }
);

export default function MapClientLoader(props: MapProps) {
  return <RealEstateMap {...props} />;
}
```

```typescript
// ✅ Step 2: Server Component（page.tsx）では MapClientLoader を使う
// app/page.tsx（Server Component）
import MapClientLoader from "@/components/Map/MapClientLoader";

export default function HomePage() {
  return (
    <main>
      <MapClientLoader projects={REDEVELOPMENT_PROJECTS} />
    </main>
  );
}
```

---

## Google Maps APIの初期化

`@googlemaps/js-api-loader` を使うと、スクリプトの動的ロードが楽になる。

```bash
npm install @googlemaps/js-api-loader
```

```typescript
// components/Map/RealEstateMap.tsx
"use client";
import { useEffect, useRef } from "react";
import { Loader } from "@googlemaps/js-api-loader";

export default function RealEstateMap({ projects }: Props) {
  const mapRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const loader = new Loader({
      apiKey: process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY!,
      version: "weekly",
      libraries: ["marker"],
    });

    loader.load().then(async () => {
      const { Map } = await google.maps.importLibrary("maps") as google.maps.MapsLibrary;
      const map = new Map(mapRef.current!, {
        center: { lat: 35.6762, lng: 139.6503 },
        zoom: 12,
        mapId: process.env.NEXT_PUBLIC_GOOGLE_MAPS_ID,
      });

      // マーカーを追加
      projects.forEach((project) => {
        new google.maps.Marker({
          position: { lat: project.lat, lng: project.lng },
          map,
          title: project.projectName,
        });
      });
    });
  }, [projects]);

  return <div ref={mapRef} className="w-full h-full" />;
}
```

---

## スクリプトタグで読み込む場合（next/script）

`@googlemaps/js-api-loader` を使わずに `next/script` で読み込む方法もある。

```tsx
// app/layout.tsx
import Script from "next/script";

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Script
          src={`https://maps.googleapis.com/maps/api/js?key=${process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY}&libraries=marker`}
          strategy="afterInteractive"
        />
      </body>
    </html>
  );
}
```

この場合 `window.google` が使える状態になってから Map を初期化する必要があるため、`useEffect` の中で `window.google` の存在チェックを入れる。

---

## Advance Marker（新しいマーカーAPI）

Google Mapsの新しいマーカーAPIは `AdvancedMarkerElement` を使う。カスタムHTMLをマーカーにできる。

```typescript
const { AdvancedMarkerElement } = await google.maps.importLibrary("marker");

const markerEl = document.createElement("div");
markerEl.innerHTML = `
  <div style="background:white;border-radius:8px;padding:4px 8px;font-size:11px;font-weight:bold;border:2px solid #f43f5e">
    ${project.investmentScore}点
  </div>
`;

new AdvancedMarkerElement({
  position: { lat: project.lat, lng: project.lng },
  map,
  content: markerEl,
  title: project.projectName,
});
```

投資スコアをマーカーに直接表示することで、地図上で一覧性が高まる。

---

## よくある別のエラー

### `mapId` が必要

Advanced Markerを使う場合は `mapId` が必須。Google Cloud Console → Maps Platform → Map IDs で作成する。

```typescript
const map = new Map(el, {
  mapId: "YOUR_MAP_ID", // 必須
});
```

### Loader の二重初期化

`useEffect` の外で `new Loader()` を作ると、React の Strict Mode（開発環境）でエラーが出ることがある。`useEffect` の中でインスタンス化する。

---

## まとめ

| 問題 | 解決策 |
|------|--------|
| `ssr: false` がServer Componentで使えない | `"use client"` ラッパーを作る |
| `window is not defined` | `ssr: false` + `useEffect` 内で初期化 |
| Advanced Markerが動かない | `mapId` を設定する |
| Loaderの二重初期化 | `useEffect` 内でインスタンス化 |

PropAnalyticsのマップ実装の詳細は「[不動産投資データサービスをNext.js×Supabaseで作った全記録](https://zenn.dev/keisuke58/books/how-to-build-propanalytics)」を参照。
