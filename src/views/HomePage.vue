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
            <ion-button fill="solid" @click="presentActionSheet">+</ion-button>
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
import { ref, onMounted } from 'vue';
import { Filesystem, Directory, Encoding } from '@capacitor/filesystem';
import { FilePicker } from '@capawesome/capacitor-file-picker';
import { IonActionSheet, actionSheetController, alertController, IonButton } from '@ionic/vue';
import { FileOpener } from '@capawesome-team/capacitor-file-opener';

const fileListElement = ref<HTMLElement | null>(null);

onMounted(async () => {
  fileListElement.value = document.getElementById('file-list');
  await loadFiles();
});

async function loadFiles(directory:
    /* __placeholder__ */
    Directory=Directory.Data, name:
    /* __placeholder__ */
    string='') {
  if (!fileListElement.value) {
    return;
  }

  try {
    const result = await Filesystem.readdir({
      directory: Directory.Data,
      path: ''
    });

    if (result.files.length === 0) {
      fileListElement.value.innerHTML = '<ion-item><ion-label>Keine Dateien gefunden</ionlabel></ion-item>';
      return;
    }
    fileListElement.value.innerHTML = ''; // Clear the existing list
    result.files.forEach(file => {
      const fileItem = document.createElement('ion-item');
      fileItem.innerHTML = `<ion-label>${file.name}</ion-label>`; // Use file.name
      fileItem.addEventListener('click', () => handleItemClick(file.name, directory));
      fileListElement.value?.appendChild(fileItem); // Use optional chaining
  });

  } catch (e) {
    console.error('Unable to read dir', e);
  }
}

async function presentActionSheet() {
  const actionSheet = await actionSheetController.create({
    header: 'Wählen Sie eine Aktion',
    buttons: [
      {
        text: 'Datei hinzufügen',
        handler: selectFile
      },
      {
        text: 'Neuen Ordner erstellen',
        handler: presentCreateFolderAlert
      },
      {
        text: 'Abbrechen',
        role: 'cancel'
      }
    ]
  });
  await actionSheet.present();
}

async function selectFile() {
  try {
    const result = await FilePicker.pickFiles({
      limit: 0
    });

    if (result.files.length > 0) {
      const file = result.files[0];
      const fileData = await Filesystem.readFile({
        path: file.name,
        directory: Directory.External
      });

      if (fileData) {
        await Filesystem.writeFile({
          path: file.name,
          data: fileData.data,
          directory: Directory.Data,
          encoding: Encoding.UTF8
        });
        await loadFiles(); // Refresh the file list
      }
    }
  } catch (e) {
    console.error('File selection failed', e);
  }
}

async function presentCreateFolderAlert() {
  const alert = await alertController.create({
    header: 'Neuen Ordner erstellen',
    inputs: [
      {
        name: 'folderName',
        type: 'text',
        placeholder: 'Ordnername'
      }
    ],
    buttons: [
      {
        text: 'Abbrechen',
        role: 'cancel'
      },
      {
        text: 'Erstellen',
        handler: async (data: { folderName: string }) => {
          if (data.folderName) {
            await createFolder(data.folderName);
          }
        }
      }
    ]
  });
  await alert.present();
}

async function createFolder(folderName: string) {
  try {
    await Filesystem.mkdir({
      path: folderName,
      directory: Directory.Data,
      recursive: false
    });
    await loadFiles(); // Refresh the file list
  } catch (e) {
    console.error('Unable to create folder', e);
  }
}

async function handleItemClick(name: string, directory: Directory) {
  try {
    // Check if the item is a file or directory
    const result = await Filesystem.stat({
      path: name,
      directory : directory ?? Directory.Data
    });

    if (result.type === "directory") {
      // Load the contents of the directory
      await loadFiles(directory, name);
    } else {
      // Open the file
      await openFile(name);
    }
  } catch (e) {
    console.error('Unable to open item', e);
  }
}


async function openFile(fileName: string) {
  try {
    const path = `${Directory.Data}/${fileName}`;
    await FileOpener.openFile({
      path: path,
    });
  } catch (e) {
    console.error('Unable to open file', e);
  }
}

function getContentType(fileName: string): string {
  const extension = fileName.split('.').pop();
  switch (extension) {
    case 'pdf':
      return 'application/pdf';
    case 'txt':
      return 'text/plain';
    case 'jpg':
    case 'jpeg':
      return 'image/jpeg';
    case 'png':
      return 'image/png';
    default:
      return 'application/octet-stream';
  }
}
</script>


<style scoped>

.button-container {
  padding-left:85%;
  padding-top: 1%;
}

.list {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

</style>