<script setup>
import { ref } from 'vue';
import JSZip from 'jszip'; // Import JSZip

const list = ref([]);
const hasImage = ref(false);
const selectedFormat = ref("None");
const len = ref(0); // Make this a reactive reference

const handleFileChange = (event) => {
  list.value = [];
  len.value = event.target.files.length; // Update len to be reactive
  hasImage.value = len.value > 0;
  for (let i = 0; i < len.value; i++) {
    list.value.push({
      file: event.target.files[i],
      url: URL.createObjectURL(event.target.files[i])
    });
  }
}

const convertImages = async () => {
  if (len.value === 1) {
    // Handle single image download directly
    const item = list.value[0];
    const img = new Image();
    img.src = item.url;

    img.onload = () => {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = img.width;
      canvas.height = img.height;
      ctx.drawImage(img, 0, 0);

      const format = selectedFormat.value.toLowerCase();
      const dataURL = canvas.toDataURL(`image/${format}`);
      
      const byteString = atob(dataURL.split(',')[1]);
      const mimeString = dataURL.split(',')[0].split(':')[1].split(';')[0];
      const ab = new Uint8Array(byteString.length);
      
      for (let i = 0; i < byteString.length; i++) {
        ab[i] = byteString.charCodeAt(i);
      }
      
      const blob = new Blob([ab], { type: mimeString });

      // Create a download link for the single image
      const link = document.createElement('a');
      link.href = URL.createObjectURL(blob);
      link.download = `${item.file.name.split('.')[0]}.${format}`; // Use original name
      link.click();
    };
  } else {
    // Handle multiple images by zipping them
    const zip = new JSZip();
    const imagePromises = list.value.map((item) => {
      return new Promise((resolve) => {
        const img = new Image();
        img.src = item.url;

        img.onload = () => {
          const canvas = document.createElement('canvas');
          const ctx = canvas.getContext('2d');
          canvas.width = img.width;
          canvas.height = img.height;
          ctx.drawImage(img, 0, 0);

          const format = selectedFormat.value.toLowerCase();
          const dataURL = canvas.toDataURL(`image/${format}`);

          const byteString = atob(dataURL.split(',')[1]);
          const mimeString = dataURL.split(',')[0].split(':')[1].split(';')[0];
          const ab = new Uint8Array(byteString.length);
          
          for (let i = 0; i < byteString.length; i++) {
            ab[i] = byteString.charCodeAt(i);
          }
          
          const blob = new Blob([ab], { type: mimeString });
          
          // Add to the zip file
          zip.file(`${item.file.name.split('.')[0]}.${format}`, blob);
          
          resolve();
        };
      });
    });

    // Wait for all images to be processed
    await Promise.all(imagePromises);

    // Generate the zip file and trigger download
    zip.generateAsync({ type: 'blob' }).then((content) => {
      const link = document.createElement('a');
      link.href = URL.createObjectURL(content);
      link.download = 'converted_images.zip'; // Name the zip file
      link.click();
    });
  }
}
</script>

<template>
  <h1 class="text-center text-lg font-bold my-4">Convert Images to JPG, PNG or WEBP</h1>
  <div class="flex flex-col items-center justify-center min-h-screen bg-gray-50">
    <div class="w-3/4 flex justify-center mt-4 flex-col items-center gap-y-4">
      <input type="file" @change="handleFileChange" accept="image/*" multiple class="block w-auto text-sm text-gray-500
               file:mr-4 file:py-2 file:px-4
               file:rounded-lg file:border-0
               file:text-sm file:font-semibold
               file:bg-violet-100 file:text-blue-700
               hover:file:bg-blue-200 hover:file:cursor-pointer" />
      <select v-model="selectedFormat" class="text-sm font-semibold rounded-lg mx-4 px-2 py-2 border border-slate-300">
        <option value="None">Choose a file format</option>
        <option value="JPG">JPG</option>
        <option value="PNG">PNG</option>
        <option value="WEBP">WEBP</option>
      </select>
      <button v-if='hasImage && selectedFormat !== "None"' @click="convertImages"
        class="text-green-700 bg-green-100 hover:bg-green-200 cursor-pointer rounded-lg py-2 px-4 text-sm font-semibold">Convert & Download</button>
    </div>

    <ul class="mt-8 w-3/4 flex flex-col space-y-6">
      <li v-for="(item, index) in list" :key="index" class="flex flex-col items-center">
        <div class="w-full max-w-xs flex items-center justify-center bg-gray-100 rounded-lg shadow-md">
          <img class="w-full object-contain rounded-lg" :src="item.url" alt="Selected Image" />
        </div>
        <p class="mt-2 text-sm text-gray-700 truncate w-full text-center">{{ item.file.name }}</p>
      </li>
    </ul>
  </div>
</template>

<style scoped></style>
