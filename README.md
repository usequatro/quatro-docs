# Quatro Documentation

This is a repository to store technical documentation of Quatro.

## Technologies

### Web Client

- React (from create-react-app)
- Redux (with Redux Toolkit)
- Material-UI for the styling.
- atlassian/react-beautiful-dnd for drag and drop.
  This library is intended for simpler lists, like the task list, and has no support for free drag and drop, like the calendar view. However, tracking the cursor coordinates, we were able to support dragging tasks into the calendar. The popular alternative react-dnd doesn’t work well enough in touch screens.

### Firebase

- Firebase console for production (only core contributors): https://console.firebase.google.com/project/tasket-project/overview
- Firebase console for development (only core contributors): https://console.firebase.google.com/project/quatro-dev-88030/overview

Quatro is built on top of Firebase. We leverage:

- Firestore as database. Note that Firestore is not like any other NoSQL DB, it has specific constraints though on how to query it.
- Hosting for serving the web client and its build assets.
- Functions for serverless backend functionality.
- Storage for images.

### Desktop Client

ToDesktop: https://www.todesktop.com/

### Google APIs

Firebase is built on Google APIs, and we also leverage others like Google Calendar.

- Google API dashboard for production (only core contributors): https://console.developers.google.com/apis/dashboard?project=tasket-project
- Google API dashboard for development (only core contributors): https://console.developers.google.com/apis/dashboard?project=quatro-dev-88030

## Data Structures

- **Task** represents a single task instance, with its title, description, start date, effort and importance, among other fields.

- **Recurring Config** represents the configuration defined for a given task to repeat itself, including its period, how often or in which weekdays. It points to the task it should use.

- **Calendar** represents a connected calendar (Google Calendar, Outlook in the future).

- **User Internal Config** stores additional information for a user that doesn’t fit in Firebase’s Auth user. This information is either sensitive or internal, like refresh tokens, so it’s for usage of backend functions only.

- **User External Config** stores additional information of the user and is served to the clients.

## Recurring Tasks

### How do tasks repeat (recurring tasks)?

Instances of tasks that repeat are created when needed. It isn’t like Google Calendar, where all instances are defined already from the beginning. Instead, there’s a Firebase function scheduled to run every few minutes. It loops through the recurring configs saved in Firestore and determines if it’s time to create a new task for each one.

## Activity Tracking

* Google Analytics. For tracking page views and general traffic on the site.
* Mixpanel. For tracking and analyzing user actions at scale. We send events from frontend and backend functions.
* ActiveCampaign [🚫 inactive]. For segmented marketing to users depending on tags and custom fields tracked.


## Sign up / Log in with Firebase Auth

Firebase Auth used for signing up and logging in.

Providers supported:

- Email / password.
- Google.
    - If logging in or signing up with Google for an account managed by email / password, the email / password provider will be:
        - Preserved if the email was verified.
        - Removed if the email wasn’t verified (in the future, the user can only log in via Google).
    - When signing with Google, we actually sign in to Google API first so that we already can show calendars (when the permission is granted). After signing in to Google API, we programmatically sign in to Firebase. The user doesn’t notice.

## Google Calendar Integration

Quatro integrates with Google calendar to:

- Show events from one or more connected calendars in the Quatro interface.
- Create Google Calendar events for tasks, blocking time to do them.

Note that when making changes to the scopes, we need to update the Privacy Policy ([link](https://www.usequatro.com/privacy-policy))

### How does it work on the client?

Via the Google JavaScript Client API. The user needs to log in to their Google account, which makes Google become an auth provider to the Firebase user. In the browser, the user gets logged in to both the Firebase auth client and the Google API Auth2 client.

The Google API lib needs to be logged in using a popup, but Firebase auth can be logged in via token. Therefore, when we know they use Google as provider, we can ask the user to log in to Google API, and then log in to Firebase programmatically. ([ref](https://github.com/msukmanowsky/gapi-firebase))

If the user used the password Firebase provider, they’d log in to Firebase but not GoogleAPI. We improve this flow by linking their Google account to their existing Firebase account when connecting a calendar. Still, if a user logged in later with email/password, we can’t log in automatically to GoogleAPI, so they’d need to sign in from the web.

There is validation in the application to ensure that the Firebase and Google API accounts match.

### How does it work on the server?

Calendar events for tasks are managed by Firebase functions, not directly from the client. This is to easily centralize management of calendar events and make it client-agnostic.

We have the **syncTaskWithGoogleCalendar** trigger function triggered when creating, updating and deleting tasks that will create, update or delete Google Calendar events as needed.

We also have the **notifyGoogleCalendarChange** http function that acts as Google Calendar’s webhook. This URL is hit every time there’s a change within a subscribed calendar. Unfortunately, we don’t receive what the changes were, so we have to list them from within the function. We use data info to update calendar blocks to match the events, or clear them when the event was deleted.

### Renewing the webhook

Quatro watches connected calendars on behalf of the Google user accounts. To do this:

1. It triggers a watch call a Google Calendar when a calendar entity is created (**watchCalendar**). With this, a channel is created and its ID stored in Firestore.
2. It triggers a stop call, removing the watcher, when the calendar entity is deleted in Quatro (**unwatchCalendar**).
3. It has a recurring call to renew watchers that are about to expire (**renewGoogleCalendarWatchers**). When changes are made, we update a date in the calendar entity to notify the Quatro clients that there were updates, so they should consider their loaded calendar events stale.

### Connecting multiple calendars

An important constraint of the Google Calendar integration via Google API in the frontend is that the user can only be logged in to one Google account at once. To show multiple calendars, those calendars must be shared with that one Google account that the user is using. Quatro can’t do the aggregation of calendars automatically. ([ref](https://github.com/google/google-api-javascript-client/issues/310)).

In order to view calendars from different Google Accounts, the user would need to share their calendars with the connected Google Account. For example: a user would share their work calendar with their personal Google account, and then connect their personal Google account to Quatro, or vice versa.

## In-Product Mails (Mailgun)

Mailgun is used to:

- Trigger email sends from the Firebase functions
- Manage templates with Handlebars replacement (Mailgun also supports sending the email body on the request directly).

Emails are manually tracked in source control in this repository: https://github.com/usequatro/quatro-emails

Email assets are stored in a separate GCloud Storage bucket, to ensure they don’t get mixed up with user data and accidentally deleted.

## Desktop Application (discontinued)

The desktop application is made with ToDesktop, [https://www.todesktop.com/](https://www.todesktop.com/). It uses a super similar implementation to [Nativefier](https://github.com/nativefier/nativefier), but abstracting it from the user. What’s especially nice is that applications come already packaged with installers for different platforms and code-signed, which is otherwise a tedious process.

To download it, we use a Firebase function to track the users who do it: [https://us-central1-tasket-project.cloudfunctions.net/downloadDesktopClient](https://us-central1-tasket-project.cloudfunctions.net/downloadDesktopClient)

In order to test changes with the desktop application, you can open the dev tools from it and execute:

```js
window.location = 'https://dev.usequatro.com'
```