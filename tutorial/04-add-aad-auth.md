<!-- markdownlint-disable MD002 MD041 -->

In dieser Übung erweitern Sie die Anwendung aus der vorherigen Übung, um die Authentifizierung mit Azure AD zu unterstützen. Dies ist erforderlich, um das erforderliche OAuth-Zugriffstoken zum Aufrufen von Microsoft Graph zu erhalten. Hierzu integrieren Sie die [Microsoft-Authentifizierungsbibliothek (MSAL) für Android](https://github.com/AzureAD/microsoft-authentication-library-for-android) in die Anwendung.

1. Klicken Sie mit der rechten Maustaste auf den Ordner **Res** , und wählen Sie **neu**und dann **Android-Ressourcenverzeichnis**aus.

1. Ändern Sie den **Ressourcentyp** in, `raw` und wählen Sie **OK**aus.

1. Klicken Sie mit der rechten Maustaste auf den neuen **RAW** -Ordner, und wählen Sie **neu**und dann **Datei**aus.

1. Nennen Sie die `msal_config.json` Datei, und wählen Sie **OK**aus.

1. Fügen Sie der Datei **msal_config. JSON** Folgendes hinzu.

    ```json
    {
      "client_id" : "YOUR_APP_ID_HERE",
      "redirect_uri" : "msauth://YOUR_PACKAGE_NAME_HERE/callback",
      "broker_redirect_uri_registered": false,
      "account_mode": "SINGLE",
      "authorities" : [
        {
          "type": "AAD",
          "audience": {
            "type": "AzureADandPersonalMicrosoftAccount"
          },
          "default": true
        }
      ]
    }
    ```

    Ersetzen `YOUR_APP_ID_HERE` Sie durch die APP-ID aus Ihrer APP-Registrierung `YOUR_PACKAGE_NAME_HERE` , und ersetzen Sie durch den Paketnamen Ihres Projekts.

> [!IMPORTANT]
> Wenn Sie die Quellcodeverwaltung wie git verwenden, wäre es jetzt ein guter Zeitpunkt, die Datei `msal_config.json` aus der Quellcodeverwaltung auszuschließen, um unbeabsichtigtes Auslaufen ihrer APP-ID zu vermeiden.

## <a name="implement-sign-in"></a>Implementieren der Anmeldung

In diesem Abschnitt aktualisieren Sie das Manifest so, dass MSAL einen Browser zur Authentifizierung des Benutzers verwenden, den Umleitungs-URI so registrieren kann, dass er von der APP verarbeitet wird, eine Authentifizierungs Hilfsklasse erstellt und die APP zur Anmeldung und Abmeldung aktualisiert.

1. Erweitern Sie den Ordner **App/Manifests** , und öffnen Sie **AndroidManifest. XML**. Fügen Sie die folgenden Elemente oberhalb `application` des-Elements hinzu.

    ```xml
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    ```

    > [!NOTE]
    > Diese Berechtigungen sind erforderlich, damit die MSAL-Bibliothek den Benutzer authentifizieren muss.

1. Fügen Sie das folgende Element innerhalb `application` des-Elements hinzu `YOUR_PACKAGE_NAME_HERE` , und ersetzen Sie die Zeichenfolge durch ihren Paketnamen.

    ```xml
    <!--Intent filter to capture authorization code response from the default browser on the
        device calling back to the app after interactive sign in -->
    <activity
        android:name="com.microsoft.identity.client.BrowserTabActivity">
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
            <data
                android:scheme="msauth"
                android:host="YOUR_PACKAGE_NAME_HERE"
                android:path="/callback" />
        </intent-filter>
    </activity>
    ```

1. Klicken Sie mit der rechten Maustaste auf den Ordner **App/Java/com. example. graphtutorial** , und wählen Sie dann **neu**und dann **Java-Klasse**aus. Nennen Sie die `AuthenticationHelper` Klasse, und wählen Sie **OK**aus.

1. Öffnen Sie die neue Datei, und ersetzen Sie den Inhalt durch Folgendes.

    ```java
    package com.example.graphtutorial;

    import android.app.Activity;
    import android.content.Context;
    import android.util.Log;
    import com.microsoft.identity.client.AuthenticationCallback;
    import com.microsoft.identity.client.IPublicClientApplication;
    import com.microsoft.identity.client.ISingleAccountPublicClientApplication;
    import com.microsoft.identity.client.PublicClientApplication;
    import com.microsoft.identity.client.exception.MsalException;

    // Singleton class - the app only needs a single instance
    // of PublicClientApplication
    public class AuthenticationHelper {
        private static AuthenticationHelper INSTANCE = null;
        private ISingleAccountPublicClientApplication mPCA = null;
        private String[] mScopes = { "User.Read", "Calendars.Read" };

        private AuthenticationHelper(Context ctx) {
            PublicClientApplication.createSingleAccountPublicClientApplication(ctx, R.raw.msal_config,
                new IPublicClientApplication.ISingleAccountApplicationCreatedListener() {
                    @Override
                    public void onCreated(ISingleAccountPublicClientApplication application) {
                        mPCA = application;
                    }

                    @Override
                    public void onError(MsalException exception) {
                        Log.e("AUTHHELPER", "Error creating MSAL application", exception);
                    }
                });
        }

        public static synchronized AuthenticationHelper getInstance(Context ctx) {
            if (INSTANCE == null) {
                INSTANCE = new AuthenticationHelper(ctx);
            }

            return INSTANCE;
        }

        // Version called from fragments. Does not create an
        // instance if one doesn't exist
        public static synchronized AuthenticationHelper getInstance() {
            if (INSTANCE == null) {
                throw new IllegalStateException(
                    "AuthenticationHelper has not been initialized from MainActivity");
            }

            return INSTANCE;
        }

        public void acquireTokenInteractively(Activity activity, AuthenticationCallback callback) {
            mPCA.signIn(activity, null, mScopes, callback);
        }

        public void acquireTokenSilently(AuthenticationCallback callback) {
            // Get the authority from MSAL config
            String authority = mPCA.getConfiguration().getDefaultAuthority().getAuthorityURL().toString();
            mPCA.acquireTokenSilentAsync(mScopes, authority, callback);
        }

        public void signOut() {
            mPCA.signOut(new ISingleAccountPublicClientApplication.SignOutCallback() {
                @Override
                public void onSignOut() {
                    Log.d("AUTHHELPER", "Signed out");
                }

                @Override
                public void onError(@NonNull MsalException exception) {
                    Log.d("AUTHHELPER", "MSAL error signing out", exception);
                }
            });
        }
    }
    ```

1. Öffnen Sie **mainrate** , und fügen `import` Sie die folgenden Anweisungen hinzu.

    ```java
    import android.util.Log;

    import com.microsoft.identity.client.AuthenticationCallback;
    import com.microsoft.identity.client.IAuthenticationResult;
    import com.microsoft.identity.client.exception.MsalClientException;
    import com.microsoft.identity.client.exception.MsalException;
    import com.microsoft.identity.client.exception.MsalServiceException;
    import com.microsoft.identity.client.exception.MsalUiRequiredException;
    ```

1. Fügen Sie der `MainActivity` -Klasse die folgende Elementeigenschaft hinzu.

    ```java
    private AuthenticationHelper mAuthHelper = null;
    ```

1. Fügen Sie am Ende der `onCreate` Funktion Folgendes hinzu.

    ```java
    // Get the authentication helper
    mAuthHelper = AuthenticationHelper.getInstance(getApplicationContext());
    ```

1. Fügen Sie der `MainActivity` -Klasse die folgenden Funktionen hinzu.

    ```java
    // Silently sign in - used if there is already a
    // user account in the MSAL cache
    private void doSilentSignIn() {
        mAuthHelper.acquireTokenSilently(getAuthCallback());
    }

    // Prompt the user to sign in
    private void doInteractiveSignIn() {
        mAuthHelper.acquireTokenInteractively(this, getAuthCallback());
    }

    // Handles the authentication result
    public AuthenticationCallback getAuthCallback() {
        return new AuthenticationCallback() {

            @Override
            public void onSuccess(IAuthenticationResult authenticationResult) {
                // Log the token for debug purposes
                String accessToken = authenticationResult.getAccessToken();
                Log.d("AUTH", String.format("Access token: %s", accessToken));
                hideProgressBar();

                setSignedInState(true);
                openHomeFragment(mUserName);
            }

            @Override
            public void onError(MsalException exception) {
                // Check the type of exception and handle appropriately
                if (exception instanceof MsalUiRequiredException) {
                    Log.d("AUTH", "Interactive login required");
                    doInteractiveSignIn();

                } else if (exception instanceof MsalClientException) {
                    if (exception.getErrorCode() == "no_current_account") {
                        Log.d("AUTH", "No current account, interactive login required");
                        doInteractiveSignIn();
                    } else {
                        // Exception inside MSAL, more info inside MsalError.java
                        Log.e("AUTH", "Client error authenticating", exception);
                    }
                } else if (exception instanceof MsalServiceException) {
                    // Exception when communicating with the auth server, likely config issue
                    Log.e("AUTH", "Service error authenticating", exception);
                }
                hideProgressBar();
            }

            @Override
            public void onCancel() {
                // User canceled the authentication
                Log.d("AUTH", "Authentication canceled");
                hideProgressBar();
            }
        };
    }
    ```

1. Ersetzen Sie die `signIn` vorhandenen `signOut` und-Funktionen durch Folgendes.

    ```java
    private void signIn() {
        showProgressBar();
        // Attempt silent sign in first
        // if this fails, the callback will handle doing
        // interactive sign in
        doSilentSignIn();
    }

    private void signOut() {
        mAuthHelper.signOut();

        setSignedInState(false);
        openHomeFragment(mUserName);
    }
    ```

    > [!NOTE]
    > Beachten Sie, `signIn` dass die-Methode eine unbeaufsichtigte Anmeldung (via `doSilentSignIn`) ausführt. Der Rückruf für diese Methode führt eine interaktive Anmeldung durch, wenn der stumme Fehler auftritt. Dadurch wird vermieden, dass Sie den Benutzer bei jedem Start der APP auffordern müssen.

1. Speichern Sie die Änderungen, und führen Sie die App aus.

1. Wenn Sie auf das Menüelement " **Anmelden** " tippen, wird ein Browser mit der Azure AD Anmeldeseite geöffnet. Melden Sie sich mit Ihrem Konto an.

Sobald die APP wieder aufgenommen wurde, sollte ein Zugriffstoken im Debug-Protokoll in Android Studio angezeigt werden.

![Screenshot des Logcat-Fensters in Android Studio](./images/debugger-access-token.png)

## <a name="get-user-details"></a>Abrufen von Benutzer Details

In diesem Abschnitt erstellen Sie eine Hilfsklasse für alle Aufrufe von Microsoft Graph und aktualisieren die `MainActivity` Klasse so, dass die neue Klasse verwendet wird, um den angemeldeten Benutzer abzurufen.

1. Klicken Sie mit der rechten Maustaste auf den Ordner **App/Java/com. example. graphtutorial** , und wählen Sie dann **neu**und dann **Java-Klasse**aus.

1. Nennen Sie die `GraphHelper` Klasse, und wählen Sie **OK**aus.

1. Öffnen Sie die neue Datei, und ersetzen Sie den Inhalt durch Folgendes.

    ```java
    package com.example.graphtutorial;

    import com.microsoft.graph.authentication.IAuthenticationProvider;
    import com.microsoft.graph.concurrency.ICallback;
    import com.microsoft.graph.http.IHttpRequest;
    import com.microsoft.graph.models.extensions.IGraphServiceClient;
    import com.microsoft.graph.models.extensions.User;
    import com.microsoft.graph.requests.extensions.GraphServiceClient;

    // Singleton class - the app only needs a single instance
    // of the Graph client
    public class GraphHelper implements IAuthenticationProvider {
        private static GraphHelper INSTANCE = null;
        private IGraphServiceClient mClient = null;
        private String mAccessToken = null;

        private GraphHelper() {
            mClient = GraphServiceClient.builder()
                    .authenticationProvider(this).buildClient();
        }

        public static synchronized GraphHelper getInstance() {
            if (INSTANCE == null) {
                INSTANCE = new GraphHelper();
            }

            return INSTANCE;
        }

        // Part of the Graph IAuthenticationProvider interface
        // This method is called before sending the HTTP request
        @Override
        public void authenticateRequest(IHttpRequest request) {
            // Add the access token in the Authorization header
            request.addHeader("Authorization", "Bearer " + mAccessToken);
        }

        public void getUser(String accessToken, ICallback<User> callback) {
            mAccessToken = accessToken;

            // GET /me (logged in user)
            mClient.me().buildRequest().get(callback);
        }
    }
    ```

    > [!NOTE]
    > Überprüfen Sie die Funktionsweise dieses Codes.
    >
    > - Sie implementiert die `IAuthenticationProvider` Schnittstelle, um das Zugriffstoken in `Authorization` den Header in ausgehenden HTTP-Anforderungen einzufügen.
    > - Es macht eine `getUser` Funktion verfügbar, um die Informationen des angemeldeten Benutzers vom `/me` Graph-Endpunkt abzurufen.

1. Fügen Sie die `import` folgenden Anweisungen am Anfang der **Main** -Datei hinzu.

    ```java
    import com.microsoft.graph.concurrency.ICallback;
    import com.microsoft.graph.core.ClientException;
    import com.microsoft.graph.models.extensions.IGraphServiceClient;
    import com.microsoft.graph.models.extensions.User;
    ```

1. Fügen Sie der `MainActivity` Klasse die folgende Funktion hinzu, um `ICallback` einen für den Graph-Aufruf zu generieren.

    ```java
    private ICallback<User> getUserCallback() {
        return new ICallback<User>() {
            @Override
            public void success(User user) {
                Log.d("AUTH", "User: " + user.displayName);

                mUserName = user.displayName;
                mUserEmail = user.mail == null ? user.userPrincipalName : user.mail;

                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        hideProgressBar();

                        setSignedInState(true);
                        openHomeFragment(mUserName);
                    }
                });

            }

            @Override
            public void failure(ClientException ex) {
                Log.e("AUTH", "Error getting /me", ex);
                mUserName = "ERROR";
                mUserEmail = "ERROR";

                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        hideProgressBar();

                        setSignedInState(true);
                        openHomeFragment(mUserName);
                    }
                });
            }
        };
    }
    ```

1. Entfernen Sie die folgenden Zeilen, die den Benutzernamen und die e-Mail-Adresse festlegen:

    ```java
    // For testing
    mUserName = "Megan Bowen";
    mUserEmail = "meganb@contoso.com";
    ```

1. Ersetzen Sie `onSuccess` die außer Kraft `AuthenticationCallback` Setzung in der durch Folgendes.

    ```java
    @Override
    public void onSuccess(IAuthenticationResult authenticationResult) {
        // Log the token for debug purposes
        String accessToken = authenticationResult.getAccessToken();
        Log.d("AUTH", String.format("Access token: %s", accessToken));

        // Get Graph client and get user
        GraphHelper graphHelper = GraphHelper.getInstance();
        graphHelper.getUser(accessToken, getUserCallback());
    }
    ```

Wenn Sie Ihre Änderungen speichern und die APP jetzt ausführen, wird die Benutzeroberfläche nach der Anmeldung mit dem Anzeigenamen und der e-Mail-Adresse des Benutzers aktualisiert.
