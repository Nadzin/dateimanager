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

      <ion-list>
        <ion-item>
          <div class="button-container">
            <ion-button fill="solid">+</ion-button>
          </div>
        </ion-item>
      </ion-list>
    </ion-content>
    <ion-footer>
    </ion-footer>
  </ion-page>
</template>

<script setup lang="ts">
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/vue';
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
}

</style>
