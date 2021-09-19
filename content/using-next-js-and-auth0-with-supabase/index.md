# Using Next.js and Auth0 with Supabase

# Overview

In this article, we are going to explore using [Next.js](https://nextjs.org/), [Auth0](https://auth0.com/) and [Supabase](https://supabase.io/) to build a classic Todo app! Each user will only be able to see their own todos, so we will need to implement authentication, authorization and a database.

This article will cover:

- configuring Auth0, Next.js and Supabase to work together seamlessly
- using the `nextjs-auth0` library for authentication
- implementing Row Level Security (RLS) and policies for authorization
- what a JWT is
- signing our own JWTs
- using a PostgreSQL Function to extract a property from a JWT

The final version of the Todo app code can be found [here](https://github.com/dijonmusters/supabase-auth0-example).

# Stack

**Next.js** is a React framework that makes building efficient webapps super easy. It also gives us the ability to write server-side logic — which we will need to ensure our application is secure — without needing to maintain our own server!

**Auth0** is an authentication and authorization solution that makes managing users and securing applications a breeze. It is an extremely battle-tested and mature solution for auth!

**Supabase** is an open source backend-as-a-service, which makes it possible to build an application in a weekend and scale to millions! It is a convenient wrapper around a collection of open source tools that enable database storage, object storage, authentication, authorization and real-time subscriptions. This article will just use database storage and authorization.

"Wait, if Supabase handles auth, why are we using Auth0?"

One of the real strengths of Supabase is the lack of vendor lockin. Maybe you already have users in Auth0? Maybe your company has a lot of experience with Auth0? Any of the Supabase components can be swapped out for a similar service, and hosted anywhere!

So let's do just that!

# Auth0

The first thing we need to do is sign up for a free account with [Auth0](https://auth0.com/). Once at the dashboard, we need to create a new `Tenant` for our project.

> A tenant is a way of isolating our users and settings from other applications we have with Auth0.

Click the name of your account in the top left, and then select `Create tenant` from the dropdown.

![Screen Shot 2021-09-06 at 2.56.23 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_2.56.23_pm.png)

Give your tenant a unique `Domain`, set the `Region` closest to you, and leave the `Environment Tag` set to `Developement`.

![Screen Shot 2021-09-06 at 2.58.04 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_2.58.04_pm.png)

> In a production application, you want your region to be as close as possible to the majority of your users.

Next, we want to create an Application. Select `Applications` > `Applications` from the sidebar menu, and click `+ Create Application`. We want to give it a name (this can be the same as the Tenant but does not have to be) and select `Regular Web Applications`. Click `Create`.

![Screen Shot 2021-09-06 at 3.07.25 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_3.07.25_pm.png)

From the application's page you are redirected to, select the `Settings` tab and scroll down to the `Application URIs` section.

Add the following:

`Allowed Callback URLs`: `http://localhost:3000/api/auth/callback`

`Allowed Logout URLs`: `http://localhost:3000`

Go to `Advanced Settings` > `OAuth` and confirm the `JSON Web Token (JWT) Signature Algorithm` is set to `RS256` and that `OIDC Conformant` is `enabled`.

Awesome! We now have an Auth0 instance configured to handle authentication for our application. So let's build an app!

Any web application framework could be used for this example. I am going to use Next.js. Not only does it give us a super efficient React application — with things like file-based routing out of the box — but also allows us to run server-side logic while building our application — with the `getStaticProps` function — and when the user requests a page — with the `getServerSideProps` function. We will need to do things like authentication server-side, but don't want the hassle of setting up, maintaining and paying for another server.

# Next.js

The fastest way to create a Next.js application is using the `create-next-app` package:

```bash
npx create-next-app supabase-auth0
```

Replace the contents of `pages/index.js` with:

```jsx
// pages/index.js
import styles from "../styles/Home.module.css";

const Index = () => {
  return (
    <div className={styles.container}>
      Working!
    </div>
  );
}

export default Index;
```

Run the project in Development mode:

```bash
npm run dev
```

And confirm it is working at `http://localhost:3000`.

# Authentication

Let's integrate the super fantastic `nextjs-auth0` package. This is a convenient wrapper around the Auth0 JS SDK, but specifically built for Next.js:

```bash
npm i @auth0/nextjs-auth0
```

Create a new folder at `pages/api/auth/` and add a file called `[...auth0].js` with the following content:

```jsx
// pages/api/auth/[...auth0].js

import { handleAuth } from '@auth0/nextjs-auth0';

export default handleAuth();
```

This is one of those awesome things `nextjs-auth0` gives us for free! Calling `handleAuth()` automatically creates a collection of convenient routes — such as `/login` and `/logout` and all the necessary logic for handling tokens and sessions — without us needing to do a thing!

Replace the contents of `pages/_app.js` with:

```jsx
// pages/_app.js

import React from 'react';
import { UserProvider } from '@auth0/nextjs-auth0';

const App = ({ Component, pageProps }) => {
  return (
    <UserProvider>
      <Component {...pageProps} />
    </UserProvider>
  );
}

export default App;
```

Create a `.env.local` file and add:

```
AUTH0_SECRET=generate-this-below
AUTH0_BASE_URL=http://localhost:3000
AUTH0_ISSUER_BASE_URL=https://<name-of-your-tenant>.au.auth0.com
AUTH0_CLIENT_ID=get-from-auth0-dashboard
AUTH0_CLIENT_SECRET=get-from-auth0-dashboard
```

Generate a secure `AUTH0_SECRET` by running:

```bash
node -e "console.log(crypto.randomBytes(32).toString('hex'))"
```

> `AUTH0_CLIENT_ID` and `AUTH0_CLIENT_SECRET` can be found at `Applications > Settings > Basic Information` in the Auth0 Dashboard.

> You will need to quit the Next.js server and re-run the npm run dev command, anytime new environment variables are added to the .env.local file

Let's update our `pages/index.js` file to add the ability to sign in and out:

```jsx
// pages/index.js

import styles from "../styles/Home.module.css";
import { useUser } from "@auth0/nextjs-auth0";

const Index = () => {
  const { user } = useUser();

  return (
    <div className={styles.container}>
			{user ? (
        Welcome {user.name}! <Link href="/api/auth/logout"><a>Logout</a></Link>
			) : (
				<Link href="/api/auth/login">
					<a>Login</a>;
				</Link>
			)}
    </div>
  );
};

export default Index;
```

We are using the `useUser()` hook to get the `user` object if they have signed in. If not, we are rendering a link to the Login page.

> Next.js' `Link` component is being used to enable client-side transitions, rather than reloading the entire page.

We can also handle the `loading` and `error` states:

```jsx
// pages/index.js

import styles from "../styles/Home.module.css";
import { useUser } from "@auth0/nextjs-auth0";

const Index = () => {
  const { user, error, isLoading } = useUser();

  if (isLoading) return <div className={styles.container}>Loading...</div>;
  if (error) return <div className={styles.container}>{error.message}</div>;

  return (
    <div className={styles.container}>
			{user ? (
        Welcome {user.name}! <Link href="/api/auth/logout"><a>Logout</a></Link>
			) : (
				<Link href="/api/auth/login">
					<a>Login</a>;
				</Link>
			)}
    </div>
  ); 
};

export default Index;
```

We need our users to be signed in to see their `todos` or add a new `todo`, so why not just protect this route — forcing the user to be signed in and automatically redirecting them to the `/login` route if they are not.

Thankfully, the `nextjs-auth0` library makes this super simple with the `withPageAuthRequired` function. We can tell Next.js to call this function on the server before rendering our page, by setting it to the `getServerSideProps` function.

Add the following to the `pages/index.js` file:

```jsx
export const getServerSideProps = withPageAuthRequired();
```

This function checks if we have a user signed in and handles redirecting them to the Login page if not. If we do have a user, it automatically passes the `user` object to our `Index` component as a prop, and since this is happening on the server before our component is rendered, we no longer need to handle loading or error states. This means we can significantly clean up our rendering logic.

This is what our entire file should look like:

```jsx
// pages/index.js

import styles from "../styles/Home.module.css";
import { withPageAuthRequired } from "@auth0/nextjs-auth0";

const Index = ({ user }) => {
  return (
    <div className={styles.container}>
      Welcome {user.name}! <Link href="/api/auth/logout"><a>Logout</a></Link>
    </div>
  );
};

export const getServerSideProps = withPageAuthRequired();

export default Index;
```

Very clean!

We want to display a list of todos as the landing page, but first we need somewhere to store them!

# Supabase

Head over to [app.supabase.io](http://app.supabase.io) and click `Sign In` to authenticate with GitHub. This will create a free Supabase account. From the dashboard, click `New project` and choose your `Organization`.

Enter a name, password and select a region geographically close to what you selected for your Auth0 region.

![Screen Shot 2021-09-06 at 9.14.20 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_9.14.20_pm.png)

> Make sure you choose a secure password, as this will be used for your PostgreSQL Database.

It will take a few minutes for Supabase to provision all the bits in the background, but this page conveniently displays all the values we need to get our Next.js app configured.

![Screen Shot 2021-09-06 at 9.14.49 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_9.14.49_pm.png)

Add these values to the `.env.local` file:

```
NEXT_PUBLIC_SUPABASE_URL=your-url
NEXT_PUBLIC_SUPABASE_KEY=your-anon-public-key
SUPABASE_SIGNING_SECRET=your-jwt-secret
```

> Prepending an environment variable with `NEXT_PUBLIC_` makes it available in the Next.js client. All other values will only be available in `getStaticProps`, `getServerSideProps` and serverless functions in the `pages/api/` directory.

> Adding new values to the `.env.local` file requires a restart of the Next.js dev server!

Hopefully that stalled you long enough and your Supabase project is ready to go!

Click the `Table editor` icon in the sidebar menu and select `+ Create a new table`.

![Screen Shot 2021-09-06 at 9.25.39 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_9.25.39_pm.png)

Create a `todo` table and add columns for `content`, `user_id` and `is_complete`.

![Screen Shot 2021-09-06 at 9.30.56 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_9.30.56_pm.png)

- `content` will be the text displayed for our todo.
- `user_id` will be the user that owns the todo.
- `is_complete` will signify whether the todo is done yet. We are setting the default value to `false`, as this is what we would assume for a new todo.

> Leave `Row Level Security` disabled for now. We will worry about this later.

Click `Insert row` to create some example `todos`.

![Screen Shot 2021-09-06 at 9.39.14 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_9.39.14_pm.png)

> We can leave `user_id` blank and the default value of `false` for `is_complete`.

Lets head back to our Next.js application and install the `supabase-js` library:

```bash
npm i @supabase/supabase-js
```

Create a new folder called `utils` and add a file called `supabase.js`:

```jsx
// utils/supabase.js

import { createClient } from "@supabase/supabase-js";

const getSupabase = () => {
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL,
    process.env.NEXT_PUBLIC_SUPABASE_KEY
  );

  return supabase;
};

export { getSupabase };
```

> We are making this a function as we will need to extend it later.

This function uses the environment variables we declared earlier to create a new Supabase client. Let's use our new client to fetch `todos` in `pages/index.js`.

> We can pass a configuration object to the `withPageAuthRequired` function, and declare our own `getServerSideProps` function, that will only run if the user is signed in.

```jsx
// pages/index.js

export const getServerSideProps = withPageAuthRequired({
  async getServerSideProps() {
    const supabase = getSupabase();

    const { data: todos } = await supabase.from("todo").select("*");

    return {
      props: { todos },
    };
  },
});
```

We need to remember to import the `getSupabase` function:

```jsx
import { getSupabase } from "../utils/supabase";
```

And now we can iterate over our `todos` and display them in our component:

```jsx
const Index = ({ todos }) => {
  return (
    <div className={styles.container}>
      {todos.map((todo) => (
        <p key={todo.id}>{todo.content}</p>
      ))}
    </div>
  );
};
```

We can also handle the case where there are no `todos` to display:

```jsx
const Index = ({ todos }) => {
  return (
    <div className={styles.container}>
      {todos?.length > 0 ? (
        todos.map((todo) => <p key={todo.id}>{todo.content}</p>)
      ) : (
        <p>You have completed all todos!</p>
      )}
    </div>
  );
};
```

> The `todos?.length` statement is using [Optional Chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining). This is a fallback in case the `todos` prop is `undefined` or `null`.

Our whole component should look something like this:

```jsx
// pages/index.js

import styles from "../styles/Home.module.css";
import { withPageAuthRequired } from "@auth0/nextjs-auth0";
import { getSupabase } from "../utils/supabase";

const Index = ({ todos }) => {
  return (
    <div className={styles.container}>
      {todos?.length > 0 ? (
        todos.map((todo) => <p key={todo.id}>{todo.content}</p>)
      ) : (
        <p>You have completed all todos!</p>
      )}
    </div>
  );
};

export const getServerSideProps = withPageAuthRequired({
  async getServerSideProps() {
    const supabase = getSupabase();

    const { data: todos } = await supabase.from("todo").select("*");

    return {
      props: { todos },
    };
  },
});

export default Index;
```

Awesome! We should now be seeing our todos in our Next.js app!

![Screen Shot 2021-09-07 at 2.14.48 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_2.14.48_pm.png)

But wait, we are seeing *all* the todos.

We only want users to see *their* todos. For this we need to implement authorization. Let's create a helper function in postgres to extract the currently logged in user from the request's JWT.

# PostgreSQL Functions

Head back to the Supabase dashboard, click `SQL` in the side panel and select `Query-1`. Add the following SQL blob and click `RUN`.

```sql
create or replace function auth.user_id() returns text as $$
  select nullif(current_setting('request.jwt.claim.userId', true), '')::text;
$$ language sql stable;
```

Okay, this one may look a little intimidating! Let's break down the bits we need to understand:

1. We are creating a new function called `user_id`.
2. The `auth.` part is just a way to namespace it as it is related to auth — convention in postgres.
3. This function will return a text value.
4. The body of the function is fetching the value from the `sub` field of the `jwt` that came along with the `request`.
5. If there is no `request.jwt.claim.sub` we are just returning an empty string — `''`.
6. `sub` is a naming convention for the ID of the user. This is the property Auth0 uses and will be unique for each of our users!

Not that scary!

> Check out this video on Postgres Functions to learn more!

Let's enable `Row Level Security` and use our new function to ensure only the users that own the `todo` can see it.

# Row Level Security

Because Supabase is just a PostgreSQL Database under the hood, we can take advantage of a killer feature - Row Level Security. This allows us to write authorization rules in the database itself, which can be much more efficient, and much much much more secure!

Head back to the Supabase dashboard and from the side panel select `Authentication` > `Policies` and click `Enable RLS` for the `todo` table.

![Screen Shot 2021-09-06 at 10.13.21 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_10.13.21_pm.png)

Now if we refresh our application we will see the empty state message.

![Screen Shot 2021-09-06 at 10.19.28 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_10.19.28_pm.png)

Where did our `todos` go?

By default, RLS will deny access to all rows. If we want a user to be able to see their `todos` we need to write a policy.

Back in the Supabase dashboard, click `New Policy`, then `Create a policy from scratch` and fill in the following:

![Screen Shot 2021-09-08 at 4.18.17 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-08_at_4.18.17_pm.png)

This might look a little unfamiliar, so let's break it down.

1. We are giving our policy a `name`. This can be anything.
2. We declare which actions we would like to enable. Since we want users to be able to read their own `todos`, we are choosing `SELECT`.
3. We need to specify a condition that can be `true` or `false`. If it evaluates to `true` the action will be allowed, otherwise it will continue to be denied.
4. We are calling our `auth.user_id()` function to get the currently logged in user's id and comparing it to the `user_id` column for this `todo`.

So basically, if the currently logged in user's `id` matches the `user_id` column for this `todo`, then they are allowed to read this `todo`.

> I recommend checking out [this video](https://www.youtube.com/watch?v=Ow_Uzedfohk) to learn more about how awesome and powerful Row Level Security is in Supabase!

Click `Review` to see the SQL that is being generated for us.

![Screen Shot 2021-09-08 at 4.19.34 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-08_at_4.19.34_pm.png)

Not even scary! It has just formatted the fields we entered into correct SQL syntax.

While we're here, let's add a policy to be able to add new `todos`. This will be an `INSERT` action:

![Screen Shot 2021-09-08 at 4.18.53 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-08_at_4.18.53_pm.png)

> While we probably want the user to be able to perform all four actions, it is good practice to specify separate policies rather than selecting `ALL`. This makes them easier to extend and remove in the future.

Our application does not yet need to be able to update or delete `todos`, therefore, we will not create policies for these actions.

> It is good security practice to enable the minimum amount of permissions for the application to function. We can easily write policies for these actions in the future, if we want to enable them.

Let's see if we can see our todos yet by refreshing our Next.js app.

![Screen Shot 2021-09-06 at 10.19.28 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_10.19.28_pm.png)

Still no todos 🙁 Did we mess something up?

No! We just need to add a little bit more glue to transform the JWT that Auth0 is giving our Next.js application to the format that Supabase is expecting.

# What is a JWT?

To understand this problem, we must first understand what a JWT is. A JWT is a way of encoding a JSON object into a big string, that we can use to send data between different services.

By default, the data in a JWT is not encrypted or private, just encoded.

> Aditya Shukla wrote [a great article](https://medium.com/swlh/the-difference-between-encoding-encryption-and-hashing-878c606a7aff) about the differences between encoding, encryption and hashing. Give it a read to learn more!

![Screen Shot 2021-09-07 at 1.53.20 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_1.53.20_pm.png)

This is an example taken from [jwt.io](http://jwt.io) — a great tool for working with JWTs. On the left we can see the JWT value. On the right we can see what each part represents when decoded. We have some header information about how the JWT was encoded, the payload of user data and a signature, that can be used to `verify` the token.

Don't put secret things in a JWT!!

The reason we can trust JWTs for authentication is that we can `sign` it using a secret value. This value is run through an algorithm with the payload data and a JWT string comes out the other side. We can use the signing secret to `verify` the JWT on our server. This ensures that the data hasn't been tinkered with in transit — if it has, the value of the JWT will be different and it will fail verification.

The only way it could be modified and pass the `verify` step is if someone has your signing secret. This would be bad!! And this is why we can only sign JWTs with our server — or in the case of Next.js, the `getStaticProps`, `getServerSideProps` or API routes in the `pages/api` directory.

Never expose the signing secret to the client!!

Okay, so now that we understand JWTs, what is the problem?

The signing secret used by Auth0 does not match Supabase's signing secret. While we're not using Supabase for authentication, it still uses the secret to verify the JWT each time we make a request for data. Neither of these services make the signing secret's value configurable.

Not a problem, we can just grab the `sub` property from Auth0 and `sign` a new token using the signing secret that Supabase is expecting!

# Signing a JWT

When working with JWTs, we definitely want to use a well trusted library! Auth0 have a very widely used and trusted one called `jsonwebtoken`.

Let's install it:

```bash
npm i jsonwebtoken
```

Now create a new file called `auth.js` in the `utils/` directory and add the following function:

```jsx
// utils/auth.js

const getToken = async (req, res) => {
	
};
```

In this function we want to:

1. get the user object from Auth0's session
2. create a new payload
3. sign a new token using Supabase's signing secret

We can use Auth0's `getSession` function to get the full user object:

```jsx
// utils/auth.js

import { getSession } from "@auth0/nextjs-auth0";

const getToken = async (req, res) => {
	const { user } = await getSession(req, res);
};
```

If we were to log this value out we would see something like this:

```jsx
{
    given_name: 'Jon',
    family_name: 'Meyers',
    nickname: 'dijonmusters',
    name: 'Jon Meyers',
    picture: 'https://lh3.googleusercontent.com/a-/AOh14GjJT6mbUv9BtWrIUo9kk_bDx-iuIE_Sh45lyG5d9Q=s96-c',
    locale: 'en-GB',
    updated_at: '2021-09-07T04:09:18.264Z',
    email: 'jon@supabase.io',
    email_verified: true,
    iss: 'https://supabase.au.auth0.com/',
    sub: 'google-oauth2|197382363852969365923',
    aud: 'c9ko31duRyNdc1IEyTFm3hYVDMtiUndl',
    iat: 1630987758,
    exp: 1631023758,
    nonce: 'ZUuHKBWBYN__Z3LbWPYNrAXj81floSFMBfq6_ECXfEw'
}
```

Now that is a lot of data about our user!

The only field our Supabase policy cares about `sub`, so let's create a new payload containing only this value:

```jsx
const payload = {
  sub: user.sub,
};
```

> It is good security practice to only give things the minimum amount of data and permissions they need to handle the task.

And finally we can sign a new token with this payload, using the Supabase signing secret:

```jsx
jwt.sign(payload, process.env.SUPABASE_SIGNING_SECRET);
```

Awesome! Our whole function should look something like this:

```jsx
// utils/auth.js

import { getSession } from "@auth0/nextjs-auth0";
import jwt from "jsonwebtoken";

const getToken = async (req, res) => {
  const { user } = await getSession(req, res);

  const payload = {
    sub: user.sub,
  };

  return jwt.sign(payload, process.env.SUPABASE_SIGNING_SECRET);
};

export { getToken };
```

Let's extend our `getSupabase` function in `utils/supabase.js` to accept an optional token parameter.

```jsx
// utils/supabase.js

import { createClient } from "@supabase/supabase-js";

const getSupabase = (token) => {
  const supabase = createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL,
    process.env.NEXT_PUBLIC_SUPABASE_KEY
  );

  if (token) {
    supabase.auth.session = () => ({
      access_token: token,
    });
  }

  return supabase;
};

export { getSupabase };
```

If we pass this function a token, it will attach it to the Supabase session, which will go along for the ride when we request data from Supabase!

Let's extend our `getServerSideProps` function in `pages/index.js` to get the token and pass it to `getSupabase`:

```jsx
// pages/index.js

export const getServerSideProps = withPageAuthRequired({
  async getServerSideProps({ req, res }) {
    const token = await getToken(req, res);
    const supabase = getSupabase(token);

    const { data: todos } = await supabase.from("todo").select("*");

    return {
      props: { todos },
    };
  },
});
```

We also need to remember to import the `getToken` function:

```jsx
import { getToken } from "../utils/auth";
```

Our whole component should look something like this:

```jsx
// pages/index.js

import styles from "../styles/Home.module.css";
import { withPageAuthRequired } from "@auth0/nextjs-auth0";
import { getSupabase } from "../utils/supabase";
import { getToken } from "../utils/auth";

const Index = ({ todos }) => {
  return (
    <div className={styles.container}>
      {todos?.length > 0 ? (
        todos.map((todo) => <p key={todo.id}>{todo.content}</p>)
      ) : (
        <p>You have completed all todos!</p>
      )}
    </div>
  );
};

export const getServerSideProps = withPageAuthRequired({
  async getServerSideProps({ req, res }) {
    const token = await getToken(req, res);
    const supabase = getSupabase(token);

    const { data: todos } = await supabase.from("todo").select("*");

    return {
      props: { todos },
    };
  },
});

export default Index;
```

And now when we refresh our application, we should finally see todos!!

![Screen Shot 2021-09-06 at 10.19.28 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-06_at_10.19.28_pm 1.png)

Nope!

But we are very close! If we look at the `Table editor` in the Supabase dashboard. Who is the user that owns the todos?

![Screen Shot 2021-09-07 at 2.43.23 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_2.43.23_pm.png)

NULL!!

So we just need to find out what our current `user_id` is and add it to those rows!

Head back to the Auth0 dashboard, click `User Management` > `Users` in the sidebar and select your user.

![Screen Shot 2021-09-07 at 2.46.24 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_2.46.24_pm.png)

The `user_id` is displayed at the top of your user's details page.

![Screen Shot 2021-09-07 at 2.46.47 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_2.46.47_pm.png)

Let's copy this value and paste it is as the `user_id` for the `todos`.

![Screen Shot 2021-09-07 at 2.49.53 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_2.49.53_pm.png)

This is a cool caption

Refresh our Next.js application!

Voilà! Todos!!!

![Screen Shot 2021-09-07 at 2.14.48 pm.png](https://raw.githubusercontent.com/dijonmusters/jonmeyers.io-blog/master/content/using-next-js-and-auth0-with-supabase/images/Screen_Shot_2021-09-07_at_2.14.48_pm 1.png)

The last thing we need to implement is actually adding a `todo`.

Let's add the form logic to our `pages/index.js` component.

```jsx
// pages/index.js

const Index = ({ todos, user }) => {
  const [content, setContent] = useState("");
  const [allTodos, setAllTodos] = useState([...todos]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const supabase = getSupabase();
    const { data } = await supabase
      .from("todo")
      .insert({ content, user_id: user.sub });
    setAllTodos([...todos, data[0]]);
    setContent("");
  };

  return (
    <div className={styles.container}>
      <form onSubmit={handleSubmit}>
        <input onChange={(e) => setContent(e.target.value)} value={content} />
        <button>Add</button>
      </form>
      {allTodos?.length > 0 ? (
        allTodos.map((todo) => <p key={todo.id}>{todo.content}</p>)
      ) : (
        <p>You have completed all todos!</p>
      )}
    </div>
  );
};
```

We now have an input field above our list of `todos`. When we click `add`, a new `todo` will be inserted into our DB. We are also setting the `user_id` column to the value of our `user.sub` so we know who the `todo` belongs to.

> `allTodos` is being introduced so we can call `setAllTodos` after inserting a new `todo`. This triggers a re-render, displaying our `todo` without a full page refresh.

But we are still missing one step!

When we call the `getSupabase()` function we are not giving it a token. Therefore, it is not wiring up our newly signed JWT to go along for the ride and will fail to insert the new `todo`.

We can't call the `getToken()` function from our client, as we would need to expose our signing secret to the front-end!

Big no no!

Since we have already created this token in `getServerSideProps`, we can simply pass it to our component as props, then provide it to the `getSupabase` function in `handleSubmit`.

This is what our final component looks like:

```jsx
// pages/index.js

import styles from "../styles/Home.module.css";
import { withPageAuthRequired } from "@auth0/nextjs-auth0";
import { getSupabase } from "../utils/supabase";
import { getToken } from "../utils/auth";
import { useState } from "react";

const Index = ({ todos, token, user }) => {
  const [content, setContent] = useState("");
  const [allTodos, setAllTodos] = useState([...todos]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    const supabase = getSupabase(token);
    const { data } = await supabase
      .from("todo")
      .insert({ content, user_id: user.sub });
    setAllTodos([...todos, data[0]]);
    setContent("");
  };

  return (
    <div className={styles.container}>
      <form onSubmit={handleSubmit}>
        <input onChange={(e) => setContent(e.target.value)} value={content} />
        <button>Add</button>
      </form>
      {allTodos?.length > 0 ? (
        allTodos.map((todo) => <p key={todo.id}>{todo.content}</p>)
      ) : (
        <p>You have completed all todos!</p>
      )}
    </div>
  );
};

export const getServerSideProps = withPageAuthRequired({
  async getServerSideProps({ req, res }) {
    const token = await getToken(req, res);
    const supabase = getSupabase(token);

    const { data: todos } = await supabase.from("todo").select("*");

    return {
      props: { todos, token },
    };
  },
});

export default Index;
```

Awesome! We now have a Next.js application using Auth0 for all things authentication, Supabase and Row Level Security for authorization. We learnt about JWTs and how to sign our own, as well as writing Postgres Functions to lookup values from the JWT payload in our database.

If you liked this article, [follow me on Twitter](https://twitter.com/_dijonmusters), [subscribe to my YouTube channel](https://www.youtube.com/channel/UCPitAIwktfCfcMR4kDWebDQ) and [check out my blog](https://jonmeyers.io/).

For all things Supabase [follow their Twitter](https://twitter.com/supabase/), [subscribe to their YouTube channel](https://www.youtube.com/channel/UCNTVzV1InxHV-YR0fSajqPQ) and [check out their blog](https://supabase.io/blog)!

Thanks for reading!