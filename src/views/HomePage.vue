<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>Dateimanager 3000</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
      </ion-header>
      <div class="button-container">
            <ion-button fill="solid">+</ion-button>
      </div>
      <div class="list">
        <ion-list id="file-list"></ion-list>
      </div>
    </ion-content>
    <ion-footer>
    </ion-footer>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/vue';

//Liste füllen
import { Filesystem, Directory } from '@capacitor/filesystem';

document.addEventListener('DOMContentLoaded', async () => {
  await loadFiles();
});

async function loadFiles() {
  try {
    const result = await Filesystem.readdir({
      directory: Directory.Documents,
      path: ''
    });

    const fileListElement = document.getElementById('file-list');

    if (result.files.length === 0) {
      fileListElement.innerHTML = '<ion-item><ion-label>Keine Dateien gefunden</ion-label></ion-item>';
      return;
    }

    result.files.forEach(file => {
      const fileItem = document.createElement('ion-item');
      fileItem.innerHTML = `<ion-label>${file}</ion-label>`;
      fileListElement.appendChild(fileItem);
    });
  } catch (e) {
    console.error('Unable to read dir', e);
  }
}
</script>

<style scoped>
#container {
  text-align: center;
  
  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;
  
  color: #8c8c8c;
  
  margin: 0;
}

#container a {
  text-decoration: none;
}

#ion-list {
  display:flex;
  justify-content:flex-end;
  align-items: right;
}

.button-container {
  justify-content: flex-end;
  padding-left: 95%;
  padding-top: 1%;
}

</style>
