<template>
  <div id="app">
    <div v-if="!m.accountId">
      <button @click="onSignInPopup()">
        サインイン（ポップアップ）
      </button>
      <button @click="onSignInRedirect()">
        サインイン（リダイレクト）
      </button>
    </div>
    <div v-else>
      こんにちは {{ m.accountName }} さん
      <button @click="onSignOutPopup()">サインアウト（ポップアップ）</button>
      <button @click="onSignOutRedirect()">サインアウト（リダイレクト）</button>
    </div>
    <button @click="onCallApi()">API呼び出し</button>
    <div>
      <textarea :value="m.apiResp" readonly> </textarea>
    </div>
  </div>
</template>

<style scoped>
textarea {
  width: 500px;
  height: 500px;
}
</style>

<script lang="ts">
import { defineComponent, reactive } from "@vue/composition-api";
import * as msal from "@azure/msal-browser";
import axios from "axios";

const settings = {
  issuerDomain: "hogehogecampaign.b2clogin.com",
  b2cDomain: "hogehogecampaign.onmicrosoft.com",
  clientId: "e3870b94-11e9-4d78-bdc3-9e822dc10a84",
  flowName: "B2C_1_SIGNUP_SIGNIN",
  apiScopeUrl:
    "https://hogehogecampaign.onmicrosoft.com/e3870b94-11e9-4d78-bdc3-9e822dc10a84/piyopiyo",
  redirectUri: "http://localhost:8080/",
  apiUrl: "http://localhost:7071/api/piyopiyo",
};

const msalConfig: msal.Configuration = {
  auth: {
    clientId: settings.clientId, // アプリケーション（クライアント）ＩＤ
    authority: `https://${settings.issuerDomain}/${settings.b2cDomain}/${settings.flowName}`,
    knownAuthorities: [settings.issuerDomain],
    redirectUri: settings.redirectUri,
  },
  cache: {
    cacheLocation: "sessionStorage", // "sessionStorage" | "localStorage"
    storeAuthStateInCookie: false,
  },
  system: {
    loggerOptions: {
      loggerCallback: (level: number, message: string, containsPii: any) => {
        if (containsPii) {
          return;
        }
        switch (level) {
          case msal.LogLevel.Error:
            console.error(message);
            return;
          case msal.LogLevel.Info:
            console.info(message);
            return;
          case msal.LogLevel.Verbose:
            console.debug(message);
            return;
          case msal.LogLevel.Warning:
            console.warn(message);
            return;
        }
      },
    },
  },
};

const authReq: msal.PopupRequest = {
  scopes: ["openid", "profile", settings.apiScopeUrl],
};

const logoffReq: msal.EndSessionPopupRequest = {
  postLogoutRedirectUri: settings.redirectUri,
  mainWindowRedirectUri: settings.redirectUri,
};

const tokenReq: msal.SilentRequest = {
  scopes: [...authReq.scopes],
  forceRefresh: false,
};

export default defineComponent({
  setup() {
    const m = reactive({
      accountId: undefined as string | undefined,
      accountName: undefined as string | undefined,
      apiResp: "",
    });

    const msalObj = new msal.PublicClientApplication(msalConfig);

    msalObj.handleRedirectPromise()
    .then((resp) => {
      if(resp){
        if (((resp.idTokenClaims as any).tfp  as string).toUpperCase() === settings.flowName.toUpperCase()) {
          setAccount(resp.account ?? undefined);
        }
      }
    });

    const signInPopup = () => {
      msalObj
        .loginPopup(authReq)
        .then((x) => {
          console.log(x);
          setAccount(x.account ?? undefined);
        })
        .catch((err) => {
          console.log(err);
        });
    };

    const signInRedirect = () => {
      msalObj
        .loginRedirect(authReq)
        .then((x) => {
          console.log(x);
        })
        .catch((err) => {
          console.log(err);
        });
    };


    const setAccount = (account?: msal.AccountInfo) => {
      m.accountId = account?.homeAccountId;
      m.accountName = account?.name;
    };

    const signOutPopup = () => {
      msalObj.logoutPopup(logoffReq)
      setAccount();
    };

    const signOutRedirect = () => {
      msalObj.logoutRedirect(logoffReq);
      setAccount();
    };

    const selectAccount = () => {
      const currentAccounts = msalObj.getAllAccounts();
      if (currentAccounts.length == 0) {
        return;
      } else if (currentAccounts.length == 1) {
        setAccount(currentAccounts[0]);
      } else {
        const accounts = currentAccounts.filter(
          (account) =>
            account.homeAccountId
              .toUpperCase()
              .includes(settings.flowName.toUpperCase()) &&
            account.idTokenClaims !== undefined &&
            ((account.idTokenClaims as any).iss as string)
              .toUpperCase()
              .includes(settings.issuerDomain.toUpperCase()) &&
            ((account.idTokenClaims as any).aud as string) ===
              msalConfig.auth.clientId
        );

        console.log(currentAccounts);
        console.log(accounts);

        if (
          accounts.every(
            (account) => account.localAccountId === accounts[0].localAccountId
          )
        ) {
          setAccount(accounts[0]);
        } else {
          signOutRedirect();
        }
      }
    };

    const getToken = (request: msal.SilentRequest) => {
      if (!m.accountId) throw new Error("accountId is undefined");
      request.account = msalObj.getAccountByHomeId(m.accountId) ?? undefined;
      return msalObj
        .acquireTokenSilent(request)
        .then((response) => {
          if (!response.accessToken || response.accessToken === "") {
            throw new msal.InteractionRequiredAuthError();
          }
          return response;
        })
        .catch((error) => {
          if (error instanceof msal.InteractionRequiredAuthError) {
            return msalObj
              .acquireTokenPopup(request)
              .then((response) => {
                console.log(response);
                return response;
              })
              .catch((error) => {
                console.log(error);
              });
          } else {
            console.log(error);
          }
        });
    };

    const callApi = (url: string, token?: string) => {
      console.log("callApi token: ", token);
      m.apiResp = "";
      const headers = token
        ? {
            Authorization: `Bearer ${token}`,
          }
        : {};

      axios({
        method: "post",
        url,
        headers,
      })
        .then((resp) => {
          m.apiResp = JSON.stringify(resp, null, 2);
          console.log(resp);
        })
        .catch((error) => {
          m.apiResp = error;
        });
    };

    selectAccount();

    return {
      m,
      onSignInPopup() {
        signInPopup();
      },
      onSignInRedirect() {
        signInRedirect();
      },
      onSignOutPopup() {
        signOutPopup();
      },
      onSignOutRedirect() {
        signOutRedirect();
      },
      onCallApi() {
        if (!m.accountId) {
          callApi(settings.apiUrl);
        } else {
          getToken({ ...tokenReq }).then((response) => {
            if (response) {
              console.log("token: ", response.accessToken);
              try {
                callApi(settings.apiUrl, response.accessToken);
              } catch (error) {
                console.log(error);
              }
            }
          });
        }
      },
    };
  },
});
</script>
