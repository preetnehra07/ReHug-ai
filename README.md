# ReHug-ai
AI app to hug your childhood self using photos.
src/
 ├─ App.tsx
 ├─ index.tsx
 ├─ types.ts
 ├─ components/ImageUploader.tsx
 ├─ services/geminiService.ts
index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>ReHug - Embrace Your Inner Child</title>

  <script src="https://cdn.tailwindcss.com"></script>

  <script type="importmap">
  {
    "imports": {
      "react": "https://esm.sh/react@19",
      "react-dom/client": "https://esm.sh/react-dom@19/client",
      "lucide-react": "https://esm.sh/lucide-react",
      "@google/genai": "https://esm.sh/@google/genai"
    }
  }
  </script>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/index.tsx"></script>
</body>
</html>
export interface ImageState {
  file: File | null;
  preview: string | null;
  base64: string | null;
}

export interface HistoryItem {
  id: string;
  imageUrl: string;
  timestamp: number;
}

export enum AppStatus {
  IDLE = "IDLE",
  GENERATING = "GENERATING",
  SUCCESS = "SUCCESS",
  ERROR = "ERROR",
}
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

const root = ReactDOM.createRoot(document.getElementById("root")!);

root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
import { GoogleGenAI } from "@google/genai";

const MODEL_NAME = "gemini-2.5-flash-image";

export async function generateReHugImage(childBase64: string, adultBase64: string) {
  const ai = new GoogleGenAI({
    apiKey: import.meta.env.VITE_GEMINI_API_KEY
  });

  const prompt = `
Generate a photorealistic image where the adult gently hugs their childhood self.
White studio background, soft lighting, natural proportions, emotional, realistic.
`;

  const response = await ai.models.generateContent({
    model: MODEL_NAME,
    contents: [{
      role: "user",
      parts: [
        { inlineData: { mimeType: "image/png", data: childBase64 } },
        { inlineData: { mimeType: "image/png", data: adultBase64 } },
        { text: prompt }
      ]
    }]
  });

  const part = response.candidates?.[0]?.content?.parts?.find(p => p.inlineData);

  if (!part) throw new Error("No image returned");

  return `data:image/png;base64,${part.inlineData.data}`;
}
import React, { useRef } from "react";
import { Upload, X } from "lucide-react";
import { ImageState } from "../types";

export default function ImageUploader({
  label,
  description,
  image,
  onImageChange,
}: {
  label: string;
  description: string;
  image: ImageState;
  onImageChange: (img: ImageState) => void;
}) {
  const inputRef = useRef<HTMLInputElement>(null);

  const handleFile = (file: File) => {
    const reader = new FileReader();
    reader.onload = () => {
      const base64 = (reader.result as string).split(",")[1];
      onImageChange({
        file,
        preview: URL.createObjectURL(file),
        base64,
      });
    };
    reader.readAsDataURL(file);
  };

  return (
    <div className="space-y-2">
      <p className="font-semibold">{label}</p>

      <div
        onClick={() => inputRef.current?.click()}
        className="h-48 border-2 border-dashed rounded-xl flex items-center justify-center cursor-pointer"
      >
        {image.preview ? (
          <img src={image.preview} className="h-full w-full object-cover rounded-xl" />
        ) : (
          <div className="text-center text-gray-500">
            <Upload className="mx-auto mb-2" />
            {description}
          </div>
        )}
      </div>

      <input
        ref={inputRef}
        type="file"
        hidden
        accept="image/*"
        onChange={(e) => e.target.files && handleFile(e.target.files[0])}
      />
    </div>
  );
}
import React, { useState } from "react";
import ImageUploader from "./components/ImageUploader";
import { ImageState, AppStatus } from "./types";
import { generateReHugImage } from "./services/geminiService";

export default function App() {
  const [child, setChild] = useState<ImageState>({ file: null, preview: null, base64: null });
  const [adult, setAdult] = useState<ImageState>({ file: null, preview: null, base64: null });
  const [result, setResult] = useState<string | null>(null);
  const [status, setStatus] = useState<AppStatus>(AppStatus.IDLE);

  const generate = async () => {
    if (!child.base64 || !adult.base64) return;

    setStatus(AppStatus.GENERATING);

    const img = await generateReHugImage(child.base64, adult.base64);

    setResult(img);
    setStatus(AppStatus.SUCCESS);
  };

  return (
    <div className="max-w-4xl mx-auto p-6 text-center">
      <h1 className="text-4xl font-bold mb-8">💛 ReHug</h1>

      <div className="grid md:grid-cols-2 gap-6 mb-6">
        <ImageUploader label="Childhood Photo" description="Upload child photo" image={child} onImageChange={setChild} />
        <ImageUploader label="Adult Photo" description="Upload adult photo" image={adult} onImageChange={setAdult} />
      </div>

      <button
        onClick={generate}
        className="bg-indigo-600 text-white px-6 py-3 rounded-xl"
      >
        {status === AppStatus.GENERATING ? "Generating..." : "Create ReHug"}
      </button>

      {result && <img src={result} className="mt-8 rounded-xl shadow-lg" />}
    </div>
  );
}
npm install
npm run dev
