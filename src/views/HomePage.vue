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
        <ion-list inset="true" lines="full" id="file-list"></ion-list>
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
const currentPath = ref<string[]>([]);  // Speichert den aktuellen Pfad als Array

onMounted(async () => {
  fileListElement.value = document.getElementById('file-list');
  await loadFiles();
});

async function loadFiles(directory: Directory = Directory.Data, name: string = '') {
  try {
    // Überprüfen, ob currentPath leer ist, und initialisieren, falls notwendig
    if (currentPath.value.length === 0) {
      currentPath.value = [''];  // Setzen des Root-Verzeichnisses
    }

    const fullPath = currentPath.value.join('/');

    const statResult = await Filesystem.stat({
      path: fullPath,
      directory
    });

    // Überprüfen, ob das fileListElement vorhanden ist
    if (!fileListElement.value) {
      console.error('fileListElement is null');
      return;  // Beenden der Funktion, wenn fileListElement null ist
    }

    // Wenn das Verzeichnis existiert, Dateien laden
    if (statResult.type === 'directory') {
      const result = await Filesystem.readdir({
        directory,
        path: fullPath
      });

      // Verarbeiten der Dateiliste
      if (result.files.length === 0) {
        fileListElement.value.innerHTML = '<ion-item><ion-label>Keine Dateien gefunden</ion-label></ion-item>';
        return;
      }

      fileListElement.value.innerHTML = '';

      for (const file of result.files) {
        const fileItem = document.createElement('ion-item');
        fileItem.setAttribute('data-file-name', file.name);

        const statFileResult = await Filesystem.stat({
          path: `${fullPath}/${file.name}`,
          directory
        });

        const iconUrl = statFileResult.type === 'directory' ? folderIcon : fileIcon;
        fileItem.innerHTML = `<img src="${iconUrl}" style="width: 40px; height: 40px;">&nbsp<ion-label>${file.name}</ion-label><img src="${deletebutton}" style="width: 40px; height: 40px; margin-left:8px;" class="delete-button">`;
        fileItem.addEventListener('click', (event) => handleItemClick(file.name, directory));

        // Hinzufügen des Listenelements zum DOM
        fileListElement.value.appendChild(fileItem);

        // Klick auf den Delete-Button behandelt das Löschen von Dateien/Ordnern
        fileItem.querySelector('.delete-button')?.addEventListener('click', async (event) => {
          event.stopPropagation(); // Verhindert das Auslösen des Events für das übergeordnete Element

          try {
            if (statFileResult.type === 'directory') {
              await Filesystem.rmdir({
                path: `${fullPath}/${file.name}`,
                directory: directory,
                recursive: true // Rekursives Löschen
              });
            } else {
              await Filesystem.deleteFile({
                path: `${fullPath}/${file.name}`,
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
    } else {
      console.error('Specified path is not a directory');
    }
  } catch (e) {
    console.error('Unable to read dir', e);

    // Bei einem Fehler zurück zum vorherigen Verzeichnis
    if (currentPath.value.length > 0) {
      currentPath.value.pop();  // Entfernt das letzte Element im Pfad
    }
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
        // Konvertiere currentPath.value (Array) in einen String, bevor es verwendet wird
        const fullPath = currentPath.value.join('/') + '/' + file.name;

        await Filesystem.writeFile({
          path: fullPath, // Datei im aktuellen Verzeichnis speichern
          data: fileData.data,
          directory: Directory.Data,
        });
        
        console.log('File written successfully');
        await loadFiles(Directory.Data, currentPath.value.join('/')); // Liste der Dateien im aktuellen Verzeichnis neu laden
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
    // Erstellen des vollständigen Pfads, indem das Array `currentPath.value` verwendet wird
    const fullPath = [...currentPath.value, folderName].join('/');

    // Erstellen des neuen Ordners im Dateisystem
    await Filesystem.mkdir({
      path: fullPath, // Ordner im aktuellen Verzeichnis erstellen
      directory: Directory.Data,
      recursive: false
    });

    // Lade die Dateien im aktuellen Verzeichnis neu, ohne den Pfad zu ändern
    await loadFiles(Directory.Data, currentPath.value.join('/')); // Übergebe den aktuellen Pfad als String
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
            const fullPath = [...currentPath.value, name].join('/');
            const statResult = await Filesystem.stat({
              path: fullPath,
              directory: directory
            });

            if (statResult.type === 'directory') {
              await Filesystem.rmdir({
                path: fullPath,
                directory: directory,
                recursive: true // Rekursives Löschen
              });
            } else {
              await Filesystem.deleteFile({
                path: fullPath,
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
    }

    // Verarbeite den Klick auf die Datei oder das Verzeichnis
    const fullPath = [...currentPath.value, name].join('/');

    const statResult = await Filesystem.stat({
      path: fullPath,
      directory: directory
    });

    if (statResult.type === 'directory') {
      currentPath.value.push(name); // Füge den Ordner zum Pfad hinzu
      await loadFiles(directory, fullPath);
    } else {
      await openFile(fullPath); // Übergebe den vollständigen Pfad zur openFile Funktion
    }
  } catch (e) {
    console.error('Unable to handle item click', e);
  }
}

async function openFile(filePath: string) {
  try {
    // Debug-Ausgabe des Pfades
    console.log('Attempting to get URI for file path:', filePath);

    // Erhalte den vollständigen URI der Datei
    const fileUriResult = await Filesystem.getUri({
      directory: Directory.Data,
      path: filePath,
    });

    const fileUri = fileUriResult.uri;
    console.log('Obtained file URI:', fileUri);

    // Überprüfen Sie den URI vor dem Öffnen
    if (!fileUri) {
      throw new Error('File URI is not defined');
    }

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
  if (currentPath.value.length === 0) {
    console.log('Bereits im Root-Verzeichnis');
    return;
  }

  currentPath.value.pop(); // Entferne das letzte Verzeichnis im Pfad
  const newPath = currentPath.value.join('/');

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