---
title: Nuxt Django app - Part 2
description: Create a dockerized Nuxt app with PWA and SSR support using a python REST API built with Django. Part 2, create the Nuxt frontend with PWA and SSR capabilities
image: "nuxt-python.png"
writtenBy: "Florian Bigot"
---

In this tutorial, we will setup a dockerized Nuxt app with PWA and SSR capabilities. This app will consume a python REST API built with Django.

This is the part 1 of the tutorial where we will setup the Nuxt frontend with PWA and SSR capabilities

[Part 1 here](https://blog.florianbgt.com/Nuxt_Django_REST_Docker_Part1)

You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-REST-Docker)

## 1) Setting up our project

To setup our project, we need to have Node install on our machine. We then run the following command.

```bash
npm create nuxt-app app
```

We have to answer the follwoing question. Here is what need to be installed for our app.

<div><blog-img src="create-nuxt-app.png" alt="Create nuxt app" width="100%" height="auto" class="shadow mb-3"/></div>

Once the command done, we can delete both the `node_modules` directory. We will not need it as we are going to dockerize the app.

```bash
rm -r app/node_modules
```

We now can dockerize our app!

To do so, we start by creating a `Dockerfile`.

```bash
touch app/Dockerfile
```

```txt
### app/Dockerfile
FROM node:16.6
WORKDIR /code
COPY package*.json /code/
RUN npm install
ENV NUXT_HOST 0.0.0.0
COPY . .
```

We then update our `docker-compose.yml` file to add our new service.

```yml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres:13.4
    volumes:
      - ./db:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
  api:
    restart: always
    build:
      context: api
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./api:/code
      - ./media:/media
    ports:
      - "8000:8000"
    depends_on:
      - db
  app:
    restart: always
    build:
      context: app
      dockerfile: Dockerfile
    command: npm run dev
    volumes:
      - ./app:/code
      - /code/node_modules
      - /code/.nuxt
    ports:
      - "3000:3000"
```

We can now spin up our containers using `docker-compose up`.

And we should see our nuxt app running at http://localhost:3000

<div><blog-img src="nuxt-landing-page.png" alt="Nuxt landing page" width="100%" height="auto" class="shadow mb-3"/></div>

**On windows, to enable on reloading you need to had the following to `nuxt.config.js`.**

```javascript
### app/nuxt.config.js
...
watchers: {
  webpack: {
    poll: true
  }
},
...
```

COngratulation, our Nuxt app is now setup!

## 2) Create layouts and some pages

Let's create the layouts of our app, as well as the home and login page.

We will use `bootstrap-vue` for styling. We do not need to install it as we already did it during our Nuxt app setup.

First, let's deleting Nuxt default page and components.

```bash
rm app/components/NuxtLogo.vue
rm app/components/Tutorial.vue
rm app/pages/index.vue
```

Now, let's create the files below.

```bash
mkdir app/layouts
touch app/components/Header.vue
touch app/components/Footer.vue
touch app/layouts/default.vue
touch app/layouts/headless.vue
touch app/layouts/error.vue
touch app/page/index.vue
touch app/page/login.vue
```

We start by creating a header component.

```vue
### app/components/header.vue
<template>
  <div>
    <div style="height: 65px" />
    <b-navbar toggleable="md" fixed="top" type="dark" variant="dark">
      <b-container>
        <b-navbar-brand to="/">
          <img
            src="/icon.png"
            height="40px"
            width="50px"
            alt="logo"
            class="d-inline-block bg-light p-1 rounded"
          />
          <span class="ml-2"> Awesome Recipes! </span>
        </b-navbar-brand>

        <b-navbar-toggle target="nav-collapse"></b-navbar-toggle>

        <b-collapse id="nav-collapse" is-nav>
          <b-navbar-nav>
            <b-nav-item to="/"> Home </b-nav-item>
            <b-nav-item to="/recipes"> Recipes </b-nav-item>
            <b-nav-item to="/profile"> Profile </b-nav-item>
            <b-nav-item @click="logout"> Logout </b-nav-item>
          </b-navbar-nav>
        </b-collapse>
      </b-container>
    </b-navbar>
  </div>
</template>

<script>
export default {
  methods: {
    async logout() {
      try {
        console.log("logout");
      } catch (err) {
        console.log(err);
      }
    }
  }
};
</script>
```

We then create a footer component

```vue
### app/components/Footer.vue
<template>
  <b-container
    fluid
    class="text-center bg-dark text-light border-top border-2 p-3"
  >
    <div>
      <strong>
        <a href="mailto: florian.bigot321@gmail.com" class="text-light">
          florian.bigot321@gmail.com
        </a>
      </strong>
    </div>
  </b-container>
</template>
```

We also create our default layout. This will be used for all our page transition.

```vue
### app/layouts/default.vue
<template>
  <div class="d-flex flex-column text-dark" style="min-height: 100vh">
    <Header />
    <b-container fluid class="bg-light py-4" style="flex: 1">
      <b-container class="px-0">
        <Nuxt />
      </b-container>
    </b-container>
    <Footer />
  </div>
</template>
```

We create another layout as well. This layout will be use for login and signup (no header and footer).

```vue
### app/layouts/headless.vue
<template>
  <div
    class="text-dark d-flex justify-content-center align-items-center"
    style="width: 100vw; height: 100vh"
  >
    <Nuxt />
  </div>
</template>
```

We create our error layout. This page will be displayed automaticaly by Nuxt on errors

```vue
### app/layouts/error.vue
<template>
  <div>
    <h1 v-if="error.statusCode === 404">
      Page Not Found ({{ error.statusCode }})
    </h1>
    <h1 v-else-if="error.statusCode === 500">
      An error occured on the server ({{ error.statusCode }})
    </h1>
    <h1 v-else>An error occured ({{ error.statusCode }})</h1>
    <div>{{ error.message }}</div>
    <NuxtLink to="/">Go back to the Home page</NuxtLink>
  </div>
</template>

<script>
export default {
  props: ["error"],
  layout: "headless"
};
</script>
```

```vue
### app/page/index.vue
<template>
  <div>
    <p><strong>Welcome to awesome recipes!</strong></p>
    <p>Start browsing awesome <NuxtLink to="/recipes">recipes</NuxtLink></p>
  </div>
</template>
```

```vue
### app/page/login.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Log In" class="bg-secondary text-light">
      <b-form @submit.prevent="login">
        <b-form-group id="email" label="Email" label-for="input-email">
          <b-form-input
            v-model="email"
            id="input-email"
            type="email"
            placeholder="Enter Email"
            autocomplete="username"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password"
            id="input-password"
            type="password"
            placeholder="Enter Password"
            autocomplete="current-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button type="submit" variant="light">Log In</b-button>
        </div>
        <div class="text-center mt-2">
          <b-link to="/password/reset" class="text-light"
            >Forgot your password</b-link
          >
          |
          <b-link to="/signup" class="text-light">Sign Up</b-link>
        </div>
      </b-form>
    </b-card>
  </div>
</template>

<script>
export default {
  layout: "headless",
  data() {
    return {
      email: "",
      password: "",
      loading: false
    };
  },
  methods: {
    async login() {
      console.log("login");
    }
  }
};
</script>
```

Our `home` and `login` page are now created! They should look like below.

<div><blog-img src="home.png" alt="home page" width="100%" height="auto" class="shadow mb-3"/></div>

<div><blog-img src="login.png" alt="login page" width="100%" height="auto" class="shadow mb-3"/></div>

## 3) Authentication

To manage authentication logic, we are going to use the `nuxt/auth` library.

To install it, run the following command.

```bash
docker-compose run app npm install --exact @nuxtjs/auth-next
```

To set up `nuxt/auth`, all we need to do is to add the following in our `nuxtconfig.js`:

```javascript
### app/nuxt.config.js
...
modules: [
  'bootstrap-vue/nuxt',
  '@nuxtjs/axios',
  '@nuxtjs/pwa',
  '@nuxtjs/auth-next',
],

router: {
  middleware: ["auth"],
},

axios: {
  progress: true,
},

publicRuntimeConfig: {
    axios: {
      browserBaseURL: "http://localhost:8000/",
    },
},

privateRuntimeConfig: {
    axios: {
      baseURL: "http://api:8000/",
    },
},

auth: {
  strategies: {
    local: {
      scheme: 'refresh',
      token: {
        property: 'access',
      },
      refreshToken: {
        property: 'refresh',
        data: 'refresh',
      },
      user: {
        property: false,
      },
      endpoints: {
        login: { url: '/token/', method: 'post' },
        refresh: { url: '/token/refresh/', method: 'post' },
        user: { url: '/user/', method: 'get' },
        logout: false
      },
    }
  },
  redirect: {
    login: '/login',
    logout: '/login',
    home: '/'
  },
},
...
```

We now can update our `login.vue` page to trigger login using `nuxt/auth`.

```javascript
### app/pages/login.vue
...
methods: {
  async login() {
    try {
      this.loading = true;
      await this.$auth.loginWith("local", {
        data: { email: this.email, password: this.password },
      });
    } catch (err) {
      console.log(err.response);
      this.$bvToast.toast(err.response.data.detail, {
        title: "Error while login",
        autoHideDelay: 5000,
        toaster: 'b-toaster-top-center',
        variant: "danger",
      });
    } finally {
      this.loading = false;
    }
  },
},
...
```

We also update our `Header.vue` component to trigger logout.

```javascript
### app/components/Header.vue
...
methods: {
  async logout() {
    try {
      this.$auth.logout()
    } catch (err) {
      console.log(err);
    }
  },
},
...
```

`nuxt/auth` is using Vuex in the background.

By default, Vuex is not enabled with Nuxt unless the `store` directory contains a file.

Thus, let's create an empty `recipes.js` file in ths store directory to enable Vuex store.
For now, this file remain empty. We will come back to this file later one in the tutorial.

```bash
touch app/store/recipe.js
```

We now can spin up our docker containers using `docker-compose up --build` and try to sign in/out users.

Upon successfull login, we will be redirected to the home page. And upon logout, we willl be redirected to the login page.

## 4) Signup

Next, let's create our Signup view!

For this, we create the `signup.vue` file in the pages directory.

```bash
touch app/pages/signup.vue
```

```vue
### app/pages/signup.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Sign Up" class="bg-secondary text-light">
      <b-form @submit.prevent="signup">
        <b-form-group id="email" label="Email" label-for="input-email">
          <b-form-input
            v-model="email"
            id="input-email"
            type="email"
            placeholder="Enter Email"
            autocomplete="username"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password"
            id="input-password"
            type="password"
            placeholder="Enter Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password2"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password2"
            id="input-password2"
            type="password"
            placeholder="Repeat Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button type="submit" variant="light">Sign Up</b-button>
        </div>
        <div class="text-center mt-2">
          <b-link to="/login" class="text-light">Already have an account? Log In instead</b-link>
        </div>
      </b-form>
    </b-card>
  </div>
</template>

<script>
export default {
  auth: "guest",
  layout: "headless",
  data() {
    return {
      email: "",
      password: "",
      password2: "",
      loading: false,
    };
  },
  methods: {
    async signup() {
      try {
        this.loading = true;
        this.$axios.setHeader("Authorization", null);
        await this.$axios.$post("signup/", {
          email: this.email,
          password: this.password,
          password2: this.password2,
        });
        await this.$auth.loginWith("local", {
          data: { email: this.email, password: this.password },
        });
      } catch (err) {
        console.log(err.response);
        this.$bvToast.toast(
          err.response.data.email ||
            err.response.data.password ||
            err.response.data.password2,
          {
            title: "Error while signup",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          }
        );
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

That is it, we can now Signup user.

Note that, user will be automatically signed in after sign up.

<div><blog-img src="signup.png" alt="signup page" width="100%" height="auto" class="shadow mb-3"/></div>

## 5) Email and password update

Let's create `profile.vue` in the page directory so users can change their email and password.

```bash
touch app/pages/profile.vue
```

```vue
### app/pages/profile.vue
<template>
  <div>
    <b-form @submit.prevent="changeEmail">
      <b-form-group id="email" label="Email" label-for="input-email">
        <b-form-input
          v-model="email.value"
          id="input-email"
          type="email"
          placeholder="Enter Email"
          autocomplete="username"
          required
          :disabled="loading || !email.edit"
        />
      </b-form-group>
      <template v-if="email.edit">
        <b-button @click="cancelEmail" variant="danger">Cancel</b-button>
        <b-button type="submit" variant="success">Validate</b-button>
      </template>
    </b-form>
    <template v-if="!email.edit">
      <b-button @click="email.edit = true">Change email</b-button>
      <b-button @click="password.edit = true">Change password</b-button>
    </template>
    <b-modal v-model="password.edit" title="Change password" hide-footer>
      <b-form @submit.prevent="changePassword">
        <b-form-group
          id="oldPassword"
          label="Old password"
          label-for="input-old-password"
        >
          <b-form-input
            v-model="password.old"
            id="input-old-password"
            type="password"
            placeholder="Enter old password"
            autocomplete="current-password"
            required
            :disabled="loading || !password.edit"
          />
        </b-form-group>
        <b-form-group
          id="newPassword"
          label="New password"
          label-for="input-new-password"
        >
          <b-form-input
            v-model="password.new"
            id="input-new-password"
            type="password"
            placeholder="Enter new password"
            autocomplete="new-password"
            required
            :disabled="loading || !password.edit"
          />
        </b-form-group>
        <b-form-group
          id="newPassword2"
          label="New password (again)"
          label-for="input-new-password2"
        >
          <b-form-input
            v-model="password.new2"
            id="input-new-password2"
            type="password"
            placeholder="Enter new password"
            autocomplete="new-password"
            required
            :disabled="loading || !password.edit"
          />
        </b-form-group>
        <b-button @click="cancelPassword" variant="danger">Cancel</b-button>
        <b-button type="submit" variant="success">Validate</b-button>
      </b-form>
    </b-modal>
  </div>
</template>

<script>
export default {
  async fetch() {
    this.email.value = this.$auth.user.email;
  },
  data: () => {
    return {
      email: { value: null, edit: false },
      password: { old: null, new: null, new2: null, edit: false },
      loading: false,
    };
  },
  methods: {
    cancelEmail() {
      this.email = { value: this.$auth.user.email, edit: false };
    },
    async changeEmail() {
      try {
        this.loading = true;
        const user = await this.$axios.$put("user/", {
          email: this.email.value,
        });
        await this.$auth.setUser(user);
        this.cancelEmail();
      } catch (err) {
        console.log(err);
        this.$bvToast.toast(err.response.data.email, {
          title: "Error while email change",
          autoHideDelay: 5000,
          toaster: "b-toaster-top-center",
          variant: "danger",
        });
      } finally {
        this.loading = false;
      }
    },
    cancelPassword() {
      this.password = { old: null, new: null, new2: null, edit: false };
    },
    async changePassword() {
      try {
        this.loading = true;
        await this.$axios.$put("password/change/", {
          old_password: this.password.old,
          password: this.password.new,
          password2: this.password.new2,
        });
        this.cancelPassword();
      } catch (err) {
        console.log(err);
        this.$bvToast.toast(
          err.response.data.old_password
            ? err.response.data.old_password.old_password
            : undefined ||
                err.response.data.password ||
                err.response.data.password2,
          {
            title: "Error while password change",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          }
        );
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

Now user can update their email and change their password.

<div><blog-img src="profile.png" alt="profile page" width="100%" height="auto" class="shadow mb-3"/></div>

## 6) Password reset

For password reset, we create the following files in the pages directory.

```bash
touch app/pages/password/reset/index.vue
touch app/pages/password/reset/confirm.vue
```

```vue
### app/pages/password/reset/index.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Password reset" class="bg-secondary text-light">
      <b-form v-if="!sent" @submit.prevent="resetPassword">
        <b-form-group id="email" label="Email" label-for="input-email">
          <b-form-input
            v-model="email"
            id="input-email"
            type="email"
            placeholder="Enter Email"
            autocomplete="username"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button to="/login" variant="light">Back to Login</b-button>
          <b-button type="submit" variant="success">Send reset link</b-button>
        </div>
      </b-form>
      <template v-else>
        <p>An Password reset link have been sent to <strong>{{ email }}</strong></p>
        <p>
          Did not receive the link?
          <b-button @click="sent = false" size="sm" variant="light"
            >Send a new one</b-button
          >
        </p>
      </template>
    </b-card>
  </div>
</template>

<script>
export default {
  auth: "guest",
  layout: "headless",
  data() {
    return {
      email: "",
      sent: false,
      loading: false,
    };
  },
  methods: {
    async resetPassword() {
      try {
        this.loading = true;
        await this.$axios.$post("password/reset/", { email: this.email });
        this.sent = true;
      } catch (err) {
        console.log(err.response);
        this.$bvToast.toast(err.response.data.email, {
          title: "Error while requesting password reset",
          autoHideDelay: 5000,
          toaster: "b-toaster-top-center",
          variant: "danger",
        });
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

```vue
### app/pages/password/reset/confirm.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Password reset" class="bg-secondary text-light">
      <b-form v-if="tokenIsValid" @submit.prevent="resetPassword">
        <b-form-group
          id="password"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password"
            id="input-password"
            type="password"
            placeholder="Enter Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password2"
          label="Password"
          label-for="input-password2"
        >
          <b-form-input
            v-model="password2"
            id="input-password2"
            type="password"
            placeholder="Enter Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button type="submit" variant="success">Reset password</b-button>
        </div>
      </b-form>
      <template v-else>
        <p>
          You need a valid link to reset your password
          <b-button to="/password/reset" size="sm" variant="light"
            >Request one</b-button
          >
        </p>
      </template>
    </b-card>
  </div>
</template>

<script>
export default {
  auth: "guest",
  layout: "headless",
  async fetch() {
    await this.validateToken();
  },
  data() {
    return {
      password: null,
      password2: null,
      tokenIsValid: null,
      loading: false,
    };
  },
  computed: {
    token() {
      return this.$route.query.token;
    },
  },
  methods: {
    async validateToken() {
      try {
        await this.$axios.$post("password/reset/validate_token/", {
          token: this.token,
        });
        this.tokenIsValid = true;
      } catch (err) {
        console.log(err);
      }
    },
    async resetPassword() {
      try {
        this.loading = true;
        if (this.password == this.password2) {
          await this.$axios.$post("password/reset/confirm/", {
            password: this.password,
            token: this.token,
          });
          this.$router.push("/login");
        } else {
          this.$bvToast.toast("Password fields did not match", {
            title: "Error while reseting password",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          });
        }
      } catch (err) {
        console.log(err.response.status);
        this.$bvToast.toast(
          err.response.data.password || "An error occured",
          {
            title: "Error while reseting password",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          }
        );
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

Password reset is now setup. When use request a password, they will be sent a link by email that will let them reset their password!

<div><blog-img src="password-reset.png" alt="Password reset" width="100%" height="auto" class="shadow mb-3"/></div>

## 7) Recipes pages

Let's create the logic to display, add, edit and delete the content of our app!

```bash
touch app/pages/recipes/index.vue
touch app/pages/recipes/add.vue
touch app/pages/recipes/_recipe/index.vue
touch app/pages/recipes/_recipe/edit.vue
touch app/pages/recipes/_recipe/delete.vue
```


First, we are going to implement the Vuex store logic by editing the `recipes.js` inside the store directory.

```javascript
### app/store/recipes.js
import Vue from "vue";

export const state = () => ({
  recipes: [],
});

export const mutations = {
  setRecipes(state, payload) {
    state.recipes = payload;
  },

  editRecipe(state, payload) {
    const index = state.recipes.findIndex((recipe) => recipe.id === payload.id);
    Vue.set(state.recipes, index, payload);
  },

  addRecipe(state, payload) {
    state.recipes.push(payload);
  },

  addRecipe(state, payload) {
    state.recipes.push(payload);
  },

  deleteRecipe(state, payload) {
    const index = state.recipes.findIndex((recipe) => recipe.id === payload);
    state.recipes.splice(index, 1);
  },
};

export const actions = {
  async getRecipes(context) {
    const response = await this.$axios.$get("recipes/");
    if (process.server) {
      response.forEach((recipe) => {
        recipe.image = recipe.image.replace(
          this.$config.axios.baseURL,
          this.$config.axios.browserBaseURL
        );
      });
    }
    context.commit("setRecipes", response);
  },

  async addRecipe(context, payload) {
    let formData = new FormData();
    formData.append("name", payload.name);
    if (!!payload.image) {
      formData.append("image", payload.image);
    }
    formData.append("instruction", payload.instruction);
    const response = await this.$axios.$post("recipes/", formData, {
      headers: {
        "Content-Type": "multipart/form-data",
      },
    });
    context.commit("addRecipe", response);
    return response;
  },

  async editRecipe(context, payload) {
    let formData = new FormData();
    formData.append("name", payload.name);
    if (!!payload.image) {
      formData.append("image", payload.image);
    }
    formData.append("instruction", payload.instruction);
    const response = await this.$axios.$patch(
      `recipes/${payload.id}/`,
      formData,
      {
        headers: {
          "Content-Type": "multipart/form-data",
        },
      }
    );
    context.commit("editRecipe", response);
    return response;
  },

  async deleteRecipe(context, payload) {
    const response = await this.$axios.$delete(`recipes/${payload}/`);
    context.commit("deleteRecipe", payload);
    return response;
  },
};

export const getters = {
  recipes(state) {
    return state.recipes;
  },
};
```

Note that we have to convert the image url sent on the server side to use the `browserBaseURL` instead of the `baseURL`. If we do not, our image are not going to be displayed properly.

Then, we can create our pages.

```vue
### app/pages/recipes/index.vue
<template>
  <div>
    <p>
      <big><strong>Browse awesome recipes!</strong></big>
    </p>
    <b-row>
      <b-col cols="12" sm="6" md="4" lg="3" xl="2"
        ><b-button to="/recipes/add" block class="mb-2">Add a new recipe</b-button>
      </b-col>
    </b-row>
    <b-row>
      <b-col
        v-for="recipe in recipes"
        :key="recipe.id"
        sm="6"
        lg="4"
        class="mb-4"
      >
        <NuxtLink :to="`/recipes/${recipe.id}`" class="text-dark">
          <b-card :to="`/${recipe.id}`" class="h-100">
            <b-img
              thumbnail
              rounded
              :src="recipe.image"
              :alt="`image-${recipe.name}`"
              style="width: 100%; height: 150px; object-fit: cover"
            />
            <h1>{{ recipe.name }}</h1>
          </b-card>
        </NuxtLink>
      </b-col>
    </b-row>
  </div>
</template>

<script>
export default {
  async fetch() {
    await this.fetchRecipes();
  },
  computed: {
    recipes() {
      return this.$store.getters["recipes/recipes"];
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
    },
  },
};
</script>
```

```vue
### app/pages/recipes/add.vue
<template>
  <b-form @submit.prevent="addRecipe">
    <h1>Add a new recipe</h1>
    <b-form-group label="Name:">
      <b-form-input v-model="name" trim required></b-form-input>
    </b-form-group>
    <b-form-group label="Photo:">
      <b-form-file v-model="image" accept="image/*" required></b-form-file>
    </b-form-group>
    <b-form-group label="Instructions:">
      <b-form-textarea
        id="textarea"
        v-model="instruction"
        placeholder="Enter instructions..."
        rows="3"
        max-rows="20"
        trim
        required
      ></b-form-textarea>
    </b-form-group>
    <b-row class="justify-content-center">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button to="/recipes" block>Cancel</b-button>
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button type="submit" block variant="success">Add</b-button>
      </b-col>
    </b-row>
  </b-form>
</template>

<script>
export default {
  data() {
    return {
      name: null,
      image: null,
      instruction: null,
    };
  },
  methods: {
    async addRecipe() {
      try {
        const recipe = await this.$store.dispatch("recipes/addRecipe", {
          name: this.name,
          image: this.image,
          instruction: this.instruction,
        });
        this.$router.push(`/recipes/${recipe.id}`);
      } catch (err) {
        console.log(err);
        this.$bvToast.toast("An error occured while adding your recipe", {
          title: "Error",
          autoHideDelay: 5000,
          appendToast: false,
          toaster: "b-toaster-top-center",
          solid: true,
          variant: "danger",
        });
      }
    },
  },
};
</script>
```

```vue
### app/pages/recipes/_recipe/index.vue
<template>
  <div>
    <b-button to="/recipes" block class="d-sm-none mb-2"
      >view all recipes</b-button
    >
    <b-img
      thumbnail
      rounded
      :src="recipe.image"
      :alt="`image-${recipe.name}`"
      style="width: 100%; height: 250px; object-fit: cover"
    />
    <h1>{{ recipe.name }}</h1>
    <b-row v-if="recipe.user.id === $auth.user.id">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}/edit`" block variant="warning"
          >Edit</b-button
        >
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}/delete`" block variant="danger"
          >Delete</b-button
        >
      </b-col>
    </b-row>
    <h2>
      <small
        >Created
        {{
          new Date(recipe.created_at).toLocaleDateString("en", {
            year: "numeric",
            month: "long",
            day: "numeric",
          })
        }}</small
      >
    </h2>
    <h2>
      <small
        >Updated
        {{
          new Date(recipe.updated_at).toLocaleDateString("en", {
            year: "numeric",
            month: "long",
            day: "numeric",
          })
        }}</small
      >
    </h2>
    <p>
      <big
        ><strong>By {{ recipe.user.email }}</strong></big
      >
    </p>
    <p style="white-space: pre-line">
      {{ recipe.instruction }}
    </p>
  </div>
</template>

<script>
export default {
  async asyncData({ params, store, error }) {
    await store.dispatch("recipes/getRecipes");
    const recipe = store.getters["recipes/recipes"].find(
      (recipe) => recipe.id === parseInt(params.recipe)
    );
    if (!recipe) {
      error({ statusCode: 404, message: "Page not found" });
      return
    }
    return recipe;
  },
  computed: {
    recipeId() {
      return parseInt(
        this.$route.path.split("/")[this.$route.path.split("/").length - 1]
      );
    },
    recipe() {
      return this.$store.getters["recipes/recipes"].find(
        (recipe) => recipe.id === this.recipeId
      );
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
    },
  },
};
</script>
```

```vue
### app/pages/recipes/_recipe/edit.vue
<template>
  <b-form @submit.prevent="editRecipe">
    <h1>Edit recipe</h1>
    <b-form-group label="Name:">
      <b-form-input
        v-model="name"
        placeholder="Enter a name ..."
        trim
        required
      ></b-form-input>
    </b-form-group>
    <b-form-group label="Photo:">
      <b-form-file v-model="image" accept="image/*"></b-form-file>
    </b-form-group>
    <b-form-group label="instruction:">
      <b-form-textarea
        id="textarea"
        v-model="instruction"
        placeholder="Enter instruction..."
        rows="3"
        max-rows="20"
        trim
        required
      ></b-form-textarea>
    </b-form-group>
    <b-row class="justify-content-center">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}`" block>Cancel</b-button>
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button type="submit" block variant="warning">Edit</b-button>
      </b-col>
    </b-row>
  </b-form>
</template>

<script>
export default {
  async asyncData({ params, store, $auth, error }) {
    await store.dispatch("recipes/getRecipes");
    const recipe = store.getters["recipes/recipes"].find(
      (recipe) => recipe.id === parseInt(params.recipe)
    );
    if (!recipe) {
      error({ statusCode: 404, message: "Page not found" });
      return
    }
    if (recipe.user.id !== $auth.user.id ) {
      error({ statusCode: 403, message: "You do not have access to this resource" });
      return
    }
    const name = recipe.name;
    const instruction = recipe.instruction;
    return {name, instruction};
  },
  data() {
    return {
      name: null,
      image: null,
      instruction: null,
    };
  },
  computed: {
    recipeId() {
      return parseInt(
        this.$route.path.split("/")[this.$route.path.split("/").length - 2]
      );
    },
    recipe() {
      return this.$store.getters["recipes/recipes"].find(
        (recipe) => recipe.id === this.recipeId
      );
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
    },
    async editRecipe() {
      try {
        await this.$store.dispatch("recipes/editRecipe", {
          id: this.recipeId,
          name: this.name,
          image: this.image,
          instruction: this.instruction,
        });
        this.$router.push(`/recipes/${this.recipeId}`);
      } catch (err) {
        console.log(err);
        this.$bvToast.toast("An error occured while editing your recipe", {
          title: "Error",
          autoHideDelay: 5000,
          appendToast: false,
          toaster: "b-toaster-top-center",
          solid: true,
          variant: "danger",
        });
      }
    },
  },
};
</script>
```

```vue
### app/pages/recipes/_recipe/delete.vue
<template>
  <b-form @submit.prevent="deleteRecipe">
    <p class="text-danger text-center">
      <big
        ><strong
          >You are about to delete this recipe, are you sure?</strong
        ></big
      >
    </p>
    <b-row class="justify-content-center">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}`" block>Cancel</b-button>
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button type="submit" block variant="danger">Delete</b-button>
      </b-col>
    </b-row>
  </b-form>
</template>

<script>
export default {
  async fetch() {
    if (
      !this.$store.getters["recipes/recipes"].find(
        (recipe) => recipe.id === this.recipeId
      )
    ) {
      await this.fetchRecipes();
    }
  },
  computed: {
    recipeId() {
      return parseInt(
        this.$route.path.split("/")[this.$route.path.split("/").length - 2]
      );
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
      if (!this.recipe) {
        this.$nuxt.error({
          statusCode: 404,
          message: `Recipe id ${this.recipeId} does not exists`,
        });
      }
    },
    async deleteRecipe() {
      try {
        await this.$store.dispatch("recipes/deleteRecipe", this.recipeId);
        this.$router.push("/recipes");
      } catch (err) {
        console.log(err);
        this.$bvToast.toast("An error occured while deleting your recipe", {
          title: "Error",
          autoHideDelay: 5000,
          appendToast: false,
          toaster: "b-toaster-top-center",
          solid: true,
          variant: "danger",
        });
      }
    },
  },
};
</script>
```

We are finaly done with the logic of our app!

You can now experiment with the app, create edit, delete new content!

## Conclusion

Our Nuxt app is now setup!

It has SSR capability which is great for SEO!

Moreover, it is PWA compatible, so user can actually install them locally on their phone and have a native app like feel!


You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-REST-Docker)

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com