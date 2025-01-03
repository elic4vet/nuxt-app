<template>
  <div class="flex justify-center h-screen items-start bg-gray-100 pt-4">
    <div class="border p-8 w-96 bg-white flex flex-col gap-8">
      <h1 class="text-xl font-medium">Christmas Greeting Card Maker</h1>
      <div class="border aspect-square banner-here relative" v-html="svg"></div>
      <form class="flex flex-col gap-4">
        <input type="text" v-model="form.name" class="px-4 py-1 border w-full" placeholder="Enter name" />
        <input type="text" v-model="form.greeting" class="px-4 py-1 border w-full" placeholder="Enter greeting" />
        <input type="file" accept=".png, .jpg, .jpeg" @change="handleFileChange" class="px-4 py-1 border w-full" />
        <small class="text-gray-400">Must be an image below 100kb, png, jpeg, or jpg formats only.</small>
        <button class="bg-indigo-500 text-white px-4 py-2" @click.prevent="downloadSvgAsJpeg(svg)">Download</button>
      </form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { createSSRApp } from "vue";
import { renderToString } from "@vue/server-renderer";
import { useLocalStorage, watchDebounced } from "@vueuse/core";
import satori from "satori";
import { html } from "satori-html";
import ChristmasCard from "./components/card.vue";

const form = useLocalStorage("app-form", {
  name: "",
  greeting: "Happy New Year",
  photo: "",
});

const svg = ref("");
const fonts = ref([]);

onMounted(async () => {
  fonts.value = await loadFonts([{ name: "InstrumentSans", url: "/fonts/InstrumentSans-Regular.ttf" }]);
  refreshGraphics();
});

watchDebounced(form, refreshGraphics, { deep: true, debounce: 500, maxWait: 1000 });

async function loadFonts(fonts) {
    return Promise.all(
        fonts.map(async (font) => {
            try {
                return {
                    ...font,
                    data: await (await fetch(font.url)).arrayBuffer(),
                };
            } catch (e) {
                console.error(`Failed to load font ${font.name}:`, e);
                return null; 
            }
        })
    ).then((fonts) => fonts.filter((f) => f)); 
}


async function refreshGraphics() {
    try {
        const content = await renderToHTML(ChristmasCard, form.value);
        const markup = html(content);
        svg.value = await satori(markup, { width: 1080, height: 1080, fonts: fonts.value });
    } catch (e) {
        console.error("Failed to refresh graphics:", e);
        svg.value = "";
    }
}

async function handleFileChange(event: Event) {
  const file = (event.target as HTMLInputElement)?.files?.[0];
  if (file && file.size > 100 * 1024) {
    alert("File size must be below 100kb"); 
    return;
}
  const reader = new FileReader();
  reader.onload = () => (form.value.photo = reader.result as string);
  reader.readAsDataURL(file);
}

async function renderToHTML(Component, props = {}) {
  return await renderToString(createSSRApp(Component, props));
}

function downloadSvgAsJpeg(svgString, filename = "image.jpeg") {
  const blob = new Blob([svgString], { type: "image/svg+xml" });
  const url = URL.createObjectURL(blob);
  const img = new Image();
  if (!svgString) {
        alert("SVG content is not available to download.");
        return;
    }
  img.onload = () => {
    const canvas = document.createElement("canvas");
    canvas.width = canvas.height = 1080;
    canvas.getContext("2d")?.drawImage(img, 0, 0, 1080, 1080);
    const link = document.createElement("a");
    link.href = canvas.toDataURL("image/jpeg");
    link.download = filename;
    link.click();
    URL.revokeObjectURL(url);
  };
  img.src = url;
}
</script>

<style>
.banner-here svg {
  width: 100%;
  height: 100%;
}

.aspect-square {
  padding-top: 100%;
}
</style>
