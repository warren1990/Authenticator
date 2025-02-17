<template>
  <div>
    <!-- File Backup -->
    <div v-show="!exportDisabled">
      <div class="text warning" v-if="!encryption.getEncryptionStatus()">
        {{ i18n.export_info }}
      </div>
      <div class="text">
        {{ i18n.backup_file_info }}
      </div>
      <div class="text warning" v-if="unsupportedAccounts">
        {{ i18n.otp_unsupported_warn }}
      </div>
      <div class="text warning" v-if="currentlyEncrypted">
        {{ i18n.phrase_incorrect_export }}
      </div>
      <a-button-link
        download="authenticator.txt"
        :href="exportOneLineOtpAuthFile"
        v-if="!unsupportedAccounts"
        >{{ i18n.download_backup }}</a-button-link
      >
      <a-button-link download="authenticator.json" :href="exportFile" v-else>{{
        i18n.download_backup
      }}</a-button-link>
      <a-button-link
        download="authenticator.json"
        :href="exportEncryptedFile"
        v-if="encryption.getEncryptionStatus()"
        >{{ i18n.download_enc_backup }}</a-button-link
      >
    </div>
    <a-button-link href="import.html">{{ i18n.import_backup }}</a-button-link>
    <br />
    <!-- 3rd Party Backup Services -->
    <div v-show="!backupDisabled">
      <div class="text">
        {{ i18n.storage_sync_info }}
      </div>
      <p></p>
      <a-button @click="showInfo('DrivePage')"> Google Drive </a-button>
      <a-button @click="showInfo('OneDrivePage')"> OneDrive </a-button>
      <a-button @click="showInfo('DropboxPage')"> Dropbox </a-button>
    </div>
  </div>
</template>
<script lang="ts">
import Vue from "vue";

export default Vue.extend({
  data: function () {
    const exportData = this.$store.state.accounts.exportData;
    const exportEncData = this.$store.state.accounts.exportEncData;
    const key = this.$store.state.accounts.key;

    return {
      unsupportedAccounts: hasUnsupportedAccounts(exportData),
      exportFile: getBackupFile(exportData),
      exportEncryptedFile: getBackupFile(exportEncData, key),
      exportOneLineOtpAuthFile: getOneLineOtpBackupFile(exportData),
    };
  },
  computed: {
    encryption: function () {
      return this.$store.state.accounts.encryption;
    },
    exportDisabled: function () {
      return this.$store.state.menu.exportDisabled;
    },
    currentlyEncrypted: function () {
      return this.$store.getters["accounts/currentlyEncrypted"];
    },
    backupDisabled: function () {
      return this.$store.getters["menu/storageArea"];
    },
  },
  methods: {
    showInfo(tab: string) {
      if (tab === "DropboxPage") {
        chrome.permissions.request(
          { origins: ["https://*.dropboxapi.com/*"] },
          async (granted) => {
            if (granted) {
              this.$store.commit("style/showInfo");
              this.$store.commit("currentView/changeView", tab);
            }
          }
        );
        return;
      } else if (tab === "DrivePage") {
        chrome.permissions.request(
          {
            origins: [
              "https://www.googleapis.com/*",
              "https://accounts.google.com/o/oauth2/revoke",
            ],
          },
          async (granted) => {
            if (granted) {
              this.$store.commit("style/showInfo");
              this.$store.commit("currentView/changeView", tab);
            }
            return;
          }
        );
        return;
      } else if (tab === "OneDrivePage") {
        chrome.permissions.request(
          {
            origins: [
              "https://graph.microsoft.com/me/*",
              "https://login.microsoftonline.com/common/oauth2/v2.0/token",
            ],
          },
          async (granted) => {
            if (granted) {
              this.$store.commit("style/showInfo");
              this.$store.commit("currentView/changeView", tab);
            }
            return;
          }
        );
        return;
      }
    },
  },
});

function hasUnsupportedAccounts(exportData: { [h: string]: OTPStorage }) {
  for (const entry of Object.keys(exportData)) {
    if (
      exportData[entry].type === "battle" ||
      exportData[entry].type === "steam"
    ) {
      return true;
    }
  }
  return false;
}

function getBackupFile(
  entryData: { [hash: string]: OTPStorage },
  key?: Object
) {
  if (key) {
    Object.assign(entryData, { key: key });
  }
  let json = JSON.stringify(entryData, null, 2);
  // for windows notepad
  json = json.replace(/\n/g, "\r\n");
  return downloadFileUrlBuilder(json);
}

function getOneLineOtpBackupFile(entryData: { [hash: string]: OTPStorage }) {
  const otpAuthLines: string[] = [];
  for (const hash of Object.keys(entryData)) {
    const otpStorage = entryData[hash];
    if (otpStorage.issuer) {
      otpStorage.issuer = removeUnsafeData(otpStorage.issuer);
    }
    if (otpStorage.account) {
      otpStorage.account = removeUnsafeData(otpStorage.account);
    }
    const label = otpStorage.issuer
      ? otpStorage.issuer + ":" + (otpStorage.account || "")
      : otpStorage.account || "";
    let type = "";
    if (otpStorage.type === "totp" || otpStorage.type === "hex") {
      type = "totp";
    } else if (otpStorage.type === "hotp" || otpStorage.type === "hhex") {
      type = "hotp";
    } else {
      continue;
    }

    const otpAuthLine =
      "otpauth://" +
      type +
      "/" +
      label +
      "?secret=" +
      otpStorage.secret +
      (otpStorage.issuer ? "&issuer=" + otpStorage.issuer : "") +
      (type === "hotp" ? "&counter=" + otpStorage.counter : "") +
      (type === "totp" && otpStorage.period
        ? "&period=" + otpStorage.period
        : "") +
      (otpStorage.digits ? "&digits=" + otpStorage.digits : "") +
      (otpStorage.algorithm ? "&algorithm=" + otpStorage.algorithm : "");

    otpAuthLines.push(otpAuthLine);
  }

  return downloadFileUrlBuilder(otpAuthLines.join("\r\n"));
}

function downloadFileUrlBuilder(content: string) {
  const blob = new Blob([content], { type: "application/octet-stream" });
  return URL.createObjectURL(blob);
}

function removeUnsafeData(data: string) {
  return encodeURIComponent(data.split("::")[0].replace(/:/g, ""));
}
</script>
