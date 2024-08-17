<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar class="toolbar">
        <ion-title style="margin-left: 22%; font-style: italic; color: blue;">Dateimanager 3000</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
      </ion-header>
      <div class="button-container">
            <ion-button fill="solid" style="margin-right: 70%;" @click="getBack">back</ion-button>
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
import { actionSheetController, alertController, IonButton } from '@ionic/vue';
import { FileOpener } from '@capawesome-team/capacitor-file-opener';
import folderIcon from './folder-icon.png';
import fileIcon from './file-icon.png';
import deletebutton from './delete-button.png'

const fileListElement = ref<HTMLElement | null>(null);
const currentPath = ref<string>('');  // Speichert den aktuellen Pfad


onMounted(async () => {
  fileListElement.value = document.getElementById('file-list');
  await loadFiles();
});

async function loadFiles(directory: Directory = Directory.Data, name: string = '') {
  currentPath.value = name;
  if (!fileListElement.value) {
    return;
  }

  try {
    const result = await Filesystem.readdir({
      directory,
      path: name
    });

    if (result.files.length === 0) {
      fileListElement.value.innerHTML = '<ion-item><ion-label>Keine Dateien gefunden</ion-label></ion-item>';
      return;
    }
    fileListElement.value.innerHTML = '';
    
    for (const file of result.files) {
      const fileItem = document.createElement('ion-item');
      fileItem.setAttribute('data-file-name', file.name); // Attribut zuweisen

      const statResult = await Filesystem.stat({
        path: `${name}/${file.name}`,
        directory
      });

      const iconUrl = statResult.type === 'directory' ? folderIcon : fileIcon;
      if (iconUrl === folderIcon) {
        fileItem.innerHTML = `<img src="${iconUrl}" style="width: 40px; height: 40px; margin-left:8px;">&nbsp<ion-label>${file.name}</ion-label><img src="${deletebutton}" style="width: 40px; height: 40px; margin-left:8px;" class="delete-button">`;
        fileItem.addEventListener('click', () => handleItemClick(file.name, directory));
        fileListElement.value?.appendChild(fileItem);
      }
      else
        fileItem.innerHTML = `<img src="${iconUrl}" style="width: 40px; height: 40px;">&nbsp<ion-label>${file.name}</ion-label><img src="${deletebutton}" style="width: 40px; height: 40px; margin-left:8px;" class="delete-button">`;
        fileItem.addEventListener('click', () => handleItemClick(file.name, directory));
        fileListElement.value?.appendChild(fileItem);

      // Klick auf das gesamte Item (außer Delete-Button) behandelt Öffnen von Dateien/Ordnern
      fileItem.addEventListener('click', (event) => handleItemClick(file.name, directory));

      // Klick auf den Delete-Button behandelt das Löschen von Dateien/Ordnern
      fileItem.querySelector('.delete-button')?.addEventListener('click', async (event) => {
        event.stopPropagation(); // Verhindert das Auslösen des Events für das übergeordnete Element

        try {
          if (statResult.type === 'directory') {
            await Filesystem.rmdir({
              path: `${name}/${file.name}`,
              directory: directory,
              recursive: true // Rekursives Löschen
            });
          } else {
            await Filesystem.deleteFile({
              path: `${name}/${file.name}`,
              directory: directory
            });
          }
          
          // Entferne das Listenelement aus dem DOM
          fileItem.remove();
        } catch (e) {
          console.error('Unable to delete item', e);
        }
      });

      fileListElement.value?.appendChild(fileItem);
    }
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

async function selectFile(): Promise<void> {
  try {
    const result = await FilePicker.pickFiles({
      limit: 0,
      readData: true
    });

    if (result.files.length > 0) {
      const file = result.files[0];
      const filePath: string | undefined = file.path;

      if (!filePath) {
        console.error('File path is undefined');
        return;
      }

      const fileData = await Filesystem.readFile({
        path: filePath
      });

      console.log('File data:', fileData);

      if (fileData) {
        await Filesystem.writeFile({
          path: `${currentPath.value}/${file.name}`, // Datei im aktuellen Verzeichnis speichern
          data: fileData.data,
          directory: Directory.Data,
        });
        
        console.log('File written successfully');
        await loadFiles(Directory.Data, currentPath.value); // Liste der Dateien im aktuellen Verzeichnis neu laden
      }
    }
  } catch (e: unknown) {
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
      path: `${currentPath.value}/${folderName}`, // Ordner im aktuellen Verzeichnis erstellen
      directory: Directory.Data,
      recursive: false
    });
    
    await loadFiles(Directory.Data, currentPath.value); // Liste der Dateien im aktuellen Verzeichnis neu laden
  } catch (e) {
    console.error('Unable to create folder', e);
  }
}

async function handleItemClick(name: string, directory: Directory) {
  try {
    const fileItem = document.querySelector(`[data-file-name="${name}"]`);
    
    if (fileItem) {
      // Prüfen, ob der Klick auf den Delete-Button war
      const deleteButton = fileItem.querySelector('.delete-button');
      
      if (deleteButton) {
        deleteButton.addEventListener('click', async (event) => {
          event.stopPropagation(); // Verhindert das Auslösen des Events für das übergeordnete Element
          
          // Lösche die Datei oder das Verzeichnis aus dem Dateisystem
          try {
            const statResult = await Filesystem.stat({
              path: `${directory}/${name}`,
              directory: directory
            });

            if (statResult.type === 'directory') {
              await Filesystem.rmdir({
                path: `${directory}/${name}`,
                directory: directory,
                recursive: true // Rekursives Löschen
              });
            } else {
              await Filesystem.deleteFile({
                path: `${directory}/${name}`,
                directory: directory
              });
            }

            // Entferne das Listenelement aus dem DOM
            fileItem.remove();
          } catch (e) {
            console.error('Unable to delete item', e);
          }
        });
      }

      // Prüfe, ob das Element ein Verzeichnis ist, und lade es bei einem Klick darauf
      const statResult = await Filesystem.stat({
        path: name,
        directory: directory
      });

      if (statResult.type === 'directory') {
        await loadFiles(directory, name);
      } else {
        const fullPath = `${name}`;
        await openFile(fullPath);
      }
    }
  } catch (e) {
    console.error('Unable to handle item click', e);
  }
}

async function openFile(fileName: string) {
  try {
    const fullPath = `${currentPath.value}/${fileName}`;

    // Erhalte den vollständigen URI der Datei
    const fileUriResult = await Filesystem.getUri({
      directory: Directory.Data,
      path: fullPath,
    });

    const fileUri = fileUriResult.uri;
    console.log('Attempting to open file at URI:', fileUri);

    // Datei öffnen mit FileOpener
    await FileOpener.openFile({
      path: fileUri,
    });

    console.log(`File opened successfully: ${fileUri}`);
  } catch (e) {
    console.error('Unable to open file', e);
  }
}

async function getBack() {
  // Überprüfen, ob wir uns nicht bereits im Root-Verzeichnis befinden
  if (currentPath.value === '') {
    console.log('Bereits im Root-Verzeichnis');
    return;
  }

  // Entferne das letzte Verzeichnis im Pfad
  const pathParts = currentPath.value.split('/');
  pathParts.pop();  // Entferne das letzte Segment
  const newPath = pathParts.join('/');

  // Lade das übergeordnete Verzeichnis
  await loadFiles(Directory.Data, newPath);
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
  padding-top: 1%;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}
.list {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

</style>