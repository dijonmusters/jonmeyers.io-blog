# Using Next.js and Auth0 with Supabase

# Overview

**Next.js** is a React framework that makes it very easy to run server-side logic, without needing to maintain a server.

**Auth0** is an authentication and authorization solution that makes managing users and securing applications a breeze!

**Supabase** is an open-source backend-as-a-service service, that makes it possible to build an application in a weekend and scale to millions.

In this tutorial, we are going to implement an authenticated, fullstack app integrating these three services. We will learn about:

- Authenticating with Auth0's Next.js library
- Signing JWTs
- Extracting values from JWT payload with PostgreSQL Functions
- Implementing authorization with Row Level Security (RLS)

People familiar with Supabase may be thinking:

"Wait, doesn't Supabase already handle auth?"

It can...

But maybe you already have users in Auth0? Maybe you have some experience with Auth0 or your company feels more confident with a robust and mature solution.

One of the real strengths of Supabase is the lack of vendor lockin. Supabase groups together a collection of open-source tools to streamline development, but at the end of the day it is just a convenience layer over a PostgreSQL Database, auth, object storage and real-time. Any of these services could be swapped out for something else, and hosted anywhere!

So let's do just that!

# Auth0

The first thing we need to do is sign up for a free account with [Auth0](https://auth0.com/). Once at the dashboard, we need to create a new Tenant for a our project.

> A tenant is just a way to isolate our users and settings from other Application we have with Auth0

Click the name of your account in the top left, and then select `Create tenant` from the dropdown.

![Screen Shot 2021-09-06 at 2.56.23 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_2.56.23_pm.png)

Give your tenant a `Domain`, set the `Region` closest to you, and leave `Environment Tag` set to `Developement`.

![Screen Shot 2021-09-06 at 2.58.04 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_2.58.04_pm.png)

> The region should be set closest to your users, but not something we need to worry about for this article

Next, we want to create an Application. Select Applications > Applications from the menu, and click `+ Create Application`. We want to give it a name (this can be the same as the Tenant) and select `Regular Web Applications`.

![Screen Shot 2021-09-06 at 3.07.25 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_3.07.25_pm.png)

Select the `Settings` tab and scroll down to the `Application URIs` section. Add:

`Allowed Callback URLs`: `http://localhost:3000/api/auth/callback`

`Allowed Logout URLs`: `http://localhost:3000`

Go to Advanced Settings > OAuth and confirm the `JSON Web Token (JWT) Signature Algorithm` is set to `RS256` and that `OIDC Conformant` is `enabled`.

Awesome! We now have an Auth0 instance configured to handle authentication for our application. So let's build an app!

We could use any web application framework for this example. I am going to use Next.js because it is awesome! But also, it not only gives us a super efficient client, but allows us to run some server-side logic - which we will need to ensure our application is secure.

# Next.js

The fastest way to create a Next.js application is using the `create-next-app` package.

```jsx
npx create-next-app supabase-auth0
```

Replace the contents of `src/pages/index.js` with:

```jsx
// src/pages/index.js
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

```jsx
npm run dev
```

And confirm it is working at `http://localhost:3000`.

# Authentication

Let's integrate the super fantastic `nextjs-auth0` package. This is a convenient wrapper around the Auth0 JS SDK, but specifically built for Next.js.

```jsx
npm i @auth0/nextjs-auth0
```

Create a new file called `[...auth0].js` in the `/pages/api/auth/` directory and add the following content:

```jsx
import { handleAuth } from '@auth0/nextjs-auth0';

export default handleAuth();
```

This is one of those awesome things the Auth0 Next.js library gives us! This automatically creates a collection of convenient routes - such as `/login` and `/logout` and all the necessary logic for handling tokens and sessions, without us needing to do a thing!

Replace the contents of `pages/_app.js` with:

```jsx
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

Create a `.env` file and add:

```
AUTH0_SECRET=generate-this-below
AUTH0_BASE_URL=http://localhost:3000
AUTH0_ISSUER_BASE_URL=https://<name-of-your-tenant>.au.auth0.com
AUTH0_CLIENT_ID=get-from-auth0-dashboard
AUTH0_CLIENT_SECRET=get-from-auth0-dashboard
```

Generate a secure `AUTH0_SECRET` by running:

```
node -e "console.log(crypto.randomBytes(32).toString('hex'))"
```

> `AUTH0_CLIENT_ID` and `AUTH0_CLIENT_SECRET` can be found at `Applications > Settings > Basic Information` in the Auth0 Dashboard.

You will need to quit the Next.js server and re-run the `npm run dev` command, anytime new environment variables are added to the `.env` file.

Let's update our `/pages/index.js` file to add the ability to sign in and out.

```jsx
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

So we are using the `useUser()` hook to get the `user` object if they have signed in. If not we are rendering a link to the login page.

> Next.js' `Link` component is being used to enable client-side transitions, rather than reloading the entire page.

We can also handle the `loading` and `error` states:

```jsx
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

It feels like asking the user to click a button to trigger the sign in flow is adding an unnecessary step. We need our users to be signed in in order to see their todo list or add new items, so why not just protect this route (forcing the user to be signed in) and automatically redirect them to the `/login` route if they are not.

Thankfully, the `nextjs-auth0` library make this super simple with the `withPageAuthRequired` function. We can tell Next.js to call this function on the server, before rendering our page by setting it to the `getServerSideProps` function.

Add the following to the `pages/index.js` file:

```jsx
export const getServerSideProps = withPageAuthRequired();
```

This function checks if we have a user signed in and handles redirecting them to the `/login` page if not. If we do have a user, it automatically passes the `user` object to our `Index` component as a prop, and since this is happening on the server before our component is rendered, we no longer need to handle loading or error states. This means we can significantly clean up our rendering logic.

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

Time to build our actual application and add the todo functionality! But first, we need somewhere to store them!

# Supabase

Head over to [app.supabase.io](http://app.supabase.io) and click `Sign In` to authenticate with GitHub. This will create a free Supabase account. Once on the dashboard, click `New project` and choose your `Organization`.

Choose a name, password and a region geographically close to what you selected for your Auth0 region.

![Screen Shot 2021-09-06 at 9.14.20 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_9.14.20_pm.png)

> Make sure you choose a secure password, as this will be used for your PostgreSQL Database

It will take a few minutes for Supabase to provision all the bits in the background, but this page conveniently displays all the values we need to get our Next.js app configured.

![Screen Shot 2021-09-06 at 9.14.49 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_9.14.49_pm.png)

Add these values to the `.env` file:

```
NEXT_PUBLIC_SUPABASE_URL=your-url
NEXT_PUBLIC_SUPABASE_KEY=your-anon-public-key
SUPABASE_SIGNING_SECRET=your-jwt-secret
```

> Prepending an environment variable with `NEXT_PUBLIC_` makes it available in the Next.js client. All other values will only be available in `getStaticProps`, `getServerSideProps` and serverless functions in the `pages/api/` directory.

> Adding new values to the `.env` file requires a restart of the Next.js dev server!

Hopefully that stalled you long enough and your Supabase project is ready to go!

Click the `Table editor` icon in the sidebar menu and select `+ Create a new table`.

![Screen Shot 2021-09-06 at 9.25.39 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_9.25.39_pm.png)

Create a `todo` table and add columns for `content`, `user_id` and `is_complete`.

![Screen Shot 2021-09-06 at 9.30.56 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_9.30.56_pm.png)

- `content` will be the content of our todo that will display in our application
- `user_id` will be the user that owns the todo
- `is_complete` will be used to signify whether the todo is done yet. We are setting the default value to `false` as this is what we would assume for a new todo

> Leave `Row Level Security` disabled for now. We will manually enable this later.

Click `Insert row` to create some example todos.

![Screen Shot 2021-09-06 at 9.39.14 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_9.39.14_pm.png)

> We can leave user_id blank and the default value for is_complete

Lets head back to our Next.js application and install the `supabase-js` library.

```bash
npm i @supabase/supabase-js
```

Create a new folder called `utils` and add a file called `supabase.js`

```jsx
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

> We are making this a function as we will need to extend this later

This function uses the environment variables we declared earlier to create a new Supabase client. Lets use our new client to fetch todos in `pages/index.js`.

> We can pass a configuration object to the `withPageAuthRequired` function, and declare our own `getServerSideProps` function that will run if a valid user exists

```jsx
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

We need to remember to import out `getSupabase` function.

```jsx
import { getSupabase } from "../utils/supabase";
```

And now we can iterate over our todos and display them in our component.

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

We can also handle the case where there are no `todos` to display.

```jsx
const Index = ({ todos }) => {
  return (
    <div className={styles.container}>
      {todos.length > 0 ? (
        todos.map((todo) => <p key={todo.id}>{todo.content}</p>)
      ) : (
        <p>You have completed all todos!</p>
      )}
    </div>
  );
};
```

Our whole component should look something like this:

```jsx
import styles from "../styles/Home.module.css";
import { withPageAuthRequired } from "@auth0/nextjs-auth0";
import { getSupabase } from "../utils/supabase";

const Index = ({ todos }) => {
  return (
    <div className={styles.container}>
      {todos.length > 0 ? (
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

Awesome! We are seeing all the todos!

But wait, we are seeing *all* the todos.

We probably only want users to see their personal todos. Let's implement some authorization!

# Row Level Security

Because Supabase is just a PostgreSQL Database under the hood, we can take advantage of a killer feature - Row Level Security. This allows us to write authorization rules in the Database itself, which can be much more efficient, and much much much more secure!

Head on back to the Supabase dashboard and from the side panel select Authentication > Policies and click `Enable RLS` for the `todo` table.

![Screen Shot 2021-09-06 at 10.13.21 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_10.13.21_pm.png)

Now if we refresh our application we will see the empty state message.

![Screen Shot 2021-09-06 at 10.19.28 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_10.19.28_pm.png)

Where did our `todos` go?

By default, RLS will deny access to all rows. If we want a user to be able to see their `todos` we need to write a policy.

Back in the Supabase click `New Policy`, then `Create a policy from scratch` and fill in the following:

![Screen Shot 2021-09-06 at 10.22.56 pm.png](Using%20Next%20js%20and%20Auth0%20with%20Supabase%20cc1866684f314f459188a4eeb53e444b/Screen_Shot_2021-09-06_at_10.22.56_pm.png)

This might look a little strange, so let's break it down.

1. We are giving our policy a `name`. This can be anything.
2. We select which actions we would like to enable. Since we want users to be able to read and write their own `todos`, we are selecting `ALL`. This enables `SELECT`, `INSERT`, `UPDATE` and `DELETE`.
3. We need to specify a condition that if `true` will allow this action, otherwise it will continue to be denied. `user_id()` is a function, which we will create soon. This will just return us the `id` for the currently logged in user. `user_id` is the column on the `todo` table.

So we are saying, if the currently logged in user's `id` matches the `user_id` column for this `todo` then they are allowed to perform read or write actions.

> I recommend checking out this video to learn more about how awesome and powerful Row Level Security is in Supabase!

Click `Review` and then `Save policy` before being overwhelmed by the SQL that it is being generated for us!

It actually isn't that bad. It is just the fields we entered! SQL isn't scary! Let's write some more advanced stuff to prove it!

# PostgreSQL Functions

We need a function to check who the currently logged in user is. This exists in the JWT token that is automatically sent along with the request to the Database.

Let's head back to the Supabase dashboard, click `SQL` from the side panel and select `Query-1`. Add the following SQL and click `RUN`.

```jsx
create or replace function auth.user_id() returns text as $$
  select nullif(current_setting('request.jwt.claim.sub', true), '')::text;
$$ language sql stable;
```

Okay, this one is a little bit more scary! But we can read through it and break down what is happening.

1. We are creating a new function
2. It is called `user_id` and is being used for auth, so it is `auth.user_id()`
3. It will return us a text value
4. Some magic that is pulling the value from the `sub` field of the `jwt` that came along with the `request`
5. `sub` is a naming convention for the ID of the user. This is the property Auth0 uses and will be unique for each of our users!

Not that scary!

Unfortunately, if we refresh our application, we are still getting the empty state for our `todos`!

Did we mess something up?

No! We just need to add a little bit more code to transform the JWT that Auth0 is giving our Next.js application to the format that Supabase is expecting.

# JWTs

- let's dig a little deeper
- briefly explain a JWT
- the problem
    - signing secret used by Auth0 does not match Supabase
    - neither are configurable atm
- implement getToken function
- extend getServerSideProps function - accept `req` and `res`
- mention why this function should only happen in `getStaticProps`, `getServerSideProps` and API routes - files in `pages/api`

TODO

- make sure there are no `/src` paths hanging around
- make sure I am happy to not include hosting section
- link each component to it's homepage when mentioned in the overview
- point to RLS video from article
- mention todo app in overview
- link to GitHub repo with Tailwind styling
- Add more links to things
- Add filenames to top of code snippet
- Problem
    - signing secret
    - signing algorithm
- Maybe write an enable GitHub auth with Auth0 blog on jonmeyers.io to point to? Maybe just make just talk about it in the video.