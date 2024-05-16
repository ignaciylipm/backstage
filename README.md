
# Instalación/Configuración de BackStage
## Documentación de referencia
* https://github.com/backstage/backstage/issues/11672
* https://github.com/backstage/backstage/issues/18386
* https://medium.com/@prassonmishra330/authentication-in-backstage-io-f28539d559db
* https://github.com/backstage/backstage/blob/master/docs/auth/add-auth-provider.md
* https://github.com/backstage/backstage/blob/master/docs/auth/index.md#adding-the-provider-to-the-sign-in-page
* https://github.com/backstage/backstage/issues/11119
* https://github.com/backstage/backstage/issues/24559
* https://github.com/backstage/backstage/issues/24195
* https://github.com/backstage/backstage/issues/24559

## Requisitos previos

### Integración con Gitlab

Para realizar la integración para la autorización de Backstage con Gitlab, se debe crear una aplicación a nivel de Gitlab. 

1. Set Application Name to backstage-dev or something along those lines.
2. The Authorization Callback URL should match the redirect URI set in Backstage.
    * Set this to http://localhost:7007/api/auth/gitlab/handler/frame for local development.
    * Set this to http://{APP_FQDN}:{APP_BACKEND_PORT}/api/auth/gitlab/handler/frame for non-local deployments.
    * Select the following scopes from the list:
      - **read_user** Grants read-only access to the authenticated user's profile through the /user API endpoint, which includes username, public email, and full name. Also grants access to read-only API endpoints under /users.
      - **read_repository** Grants read-only access to repositories on private projects using Git-over-HTTP (not using the API).
      - **write_repository** Grants read-write access to repositories on private projects using Git-over-HTTP (not using the API).
      - **openid** Grants permission to authenticate with GitLab using OpenID Connect. Also gives read-only access to the user's profile and group memberships.
      - **profile** Grants read-only access to the user's profile data using OpenID Connect.
      - **email** Grants read-only access to the user's primary email address using OpenID Connect.

### Requisito 2
### Requisito 3

## Instalación Backstage
### Parámetros
### Instalación básica
Lanzar el comando
```
npx @backstage/create-app@latest
```

### Modificación del catálogo

En el fichero app-config.yaml reemplazar la siguiente configuración
```
integrations:
  github:
    - host: github.com
      # This is a Personal Access Token or PAT from GitHub. You can find out how to generate this token, and more information
      # about setting up the GitHub integration here: https://backstage.io/docs/integrations/github/locations#configuration
    ### Example for how to add your GitHub Enterprise instance using the API:
    # - host: ghe.example.net
    #   apiBaseUrl: https://ghe.example.net/api/v3
    #   token: ${GHE_TOKEN}


catalog:
  rules:
    - allow: [Domain,Component, System, API, Resource, Location, User, Group, Service]
  locations:
    # Local example data, file locations are relative to the backend process, typically `packages/backend`
    - type: url
      target: https://github.com/ignaciylipm/yelb-catalog/blob/main/catalog-info.yaml
      rule:
        - allow: [Domain,Component, System, API, Resource, Location, User, Group, Service]
```


### Activar la autenticación con Gitlab

* Modificar el fichero app-config.yaml 

```
auth:
  environment: development
     providers:
    guest: {}
    gitlab:
      development:
        clientId: c9998321fe8cd1f918be771161e0895315c4f9a42b905de85cef6d301867cb17
        clientSecret: gloas-0f32b22f731e0d9892aecc76f10d2d042dfaa15f2d3f9a59307b9af43e0469e4
        signIn:
          resolvers:
            # typically you would pick one of these
            - resolver: usernameMatchingUserEntityName
            - resolver: emailMatchingUserEntityProfileEmail
            - resolver: emailLocalPartMatchingUserEntityName
```


* Modificar el fichero packages -> app -> src -> App.tsx
```
+ import { gitlabAuthApiRef } from '@backstage/core-plugin-api';
```
* En el fichero packages -> app -> src -> App.tsx remplazar 
```
  components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        providers={[
          'guest',
          {
            id: 'gitlab-auth-provider',
            title: 'GitLab',
            message: 'Sign in using GitLab',
            apiRef: gitlabAuthApiRef,
          },
        ]}
      />
    ),
  },
```

* Modificar el fichero index.ts
```
+ backend.add(import('@backstage/plugin-auth-backend-module-guest-provider'));
+ backend.add(import('@backstage/plugin-auth-backend-module-gitlab-provider'));

// See https://backstage.io/docs/backend-system/building-backends/migrating#the-auth-plugin
++//backend.add(import('@backstage/plugin-auth-backend-module-guest-provider'));
// See https://backstage.io/docs/auth/guest/provider
```

* Recargar de nuevo la configuración mediante
```
yarn dev
```

## Instalar Plugin de RBAC 
### Instalación de Backend

* Add the RBAC packages as dependencies to your Backstage instance:

Ejecutar el comando desde el directorio raiz de la instalación de backstage 
```
yarn workspace backend add @spotify/backstage-plugin-rbac-backend
```
* Configurar los usuarios administradores de RBAC

En el fichero app-config.yaml incluir la siguiente configuración
```
permission:
  enabled: true
  rbac:
    authorizedUsers:
      - group:default/admins
      - user:default/ignaciylipm
```

Incluir los plugins los cuales incluyen permisos en la configuración de RBAC. 

```
permission:
  enabled: true
+ permissionedPlugins:
+   - catalog
+   - scaffolder
  rbac:
    authorizedUsers:
      - group:default/admins
      - user:default/ignaciylipm
```

Añádir el siguiente plugin (New Backend System)
```
yarn workspace backend add @spotify/backstage-plugin-permission-backend-module-rbac
```

Incluir la siguiente configuración en el fichero packages/backend/src/index.ts

```
backend.add(import('@spotify/backstage-plugin-rbac-backend'));
backend.add(import('@spotify/backstage-plugin-permission-backend-module-rbac'));

```
Comentar las siguientes lineas en el fichero packages/backend/src/index.ts

```
//backend.add(
  //import('@backstage/plugin-permission-backend-module-allow-all-policy'),
//);
```

### Instalación de Frontend

Ejecutar el comando desde el directorio raiz de la instalación de backstage 
```
yarn workspace app add @spotify/backstage-plugin-rbac
```

Incluir la siguiente configuración
```
backend:
  auth:
    keys:
      - secret: ymJESaBHF0nM1mebYhiIKY0vjEWzV3k0
```
Instalar las routes de RBAC en la aplicación incluyendo la siguiente configuración en el fichero packages/app/src/App.tsx
```

import { RBACRoot } from '@spotify/backstage-plugin-rbac';

...

const routes = (
  <FlatRoutes>
      ...
    <Route path="/rbac" element={<RBACRoot />} />
  </FlatRoutes>
);
```

Incluir un sidebar que solo será visible a RBAC por lo usuarios autorizados, modificando el fichero packages/app/src/components/Root/Root.tsx
```
+ import { RBACSidebarItem } from '@spotify/backstage-plugin-rbac';

        <SidebarItem icon={SoundcheckIcon} to="soundcheck" text="Soundcheck" />
+       <RBACSidebarItem />
      </SidebarScrollWrapper>
```

Add la política de RBAC en el permission framework ajustando la siguiente configuración en el fichero packages/backend/src/plugins/permission.ts ** OMITIDO ESTE PASO **


Incluir la licencia del plugin
```
spotify:
  licenseKey: FD6129-9F16C5-53646C-D4E8F1-1D3407-V3
```


## Otros
Comprobar las versiones disponibles de backstage
```
npm view @backstage/create-app versions --json
```

Obtener los plugins instalados
```
backstage-cli info 

```
