# Authentication + Authorization

First off, let's clarify the difference! **Authentication** describes the process of determining _who a user is_, which generally refers to sign in, sign up, and session management logic. **Authorization** is the process of determining **what a given user can and cannot do**, based on their permission level, and any environment context.

## How we manage Authentication

We use [Clerk](https://clerk.com) for user authentication. Clerk is a service that provides basic cloud-hosted user management, as well as out-of-the-box tools for our sign-in and sign-up pages and one-time password authentication methods.

### One time password authentication

One-time password authentication (or OTP, as the kids call it), is an authentication strategy in which the user is sent a single use password each time they want to sign in. At Sharing Excess, we choose to _exclusively_ use OTP authentication for the following reasons:

1. **It's more secure.** This prevents users from creating insecure passwords, and also prevents credentials from being leaked in a data breach (internally or externally).
2. **It's much, much easier to manage**. We don't need to manage encoding passwords, enabling password reset flows, or even merging duplicate accounts that can come from multiple social logins. Our engineering resources are always scarce, and simple, clear solutions (even if sometimes narrow) work well for us.
3. **It bakes in validation**. No need to validate an email address - if you use a fake email, you won't be able to sign in. If you have a _typo_ in your email, you'll know right away and fix it. Win win!

### Syncing Clerk with the Sharing Excess Database

As we'll soon discuss, we manage permissions and other user information within our internal database, and therefore need to keep user data in-sync between Clerk and the Sharing Excess database.

When a user first signs up, they'll create an account via Clerk, entering their name, email, and phone number. For the moment, this data will exist _only_ in Clerk, and not yet in our database.

Inside our client applications (app.sharingexcess.com + partners.sharingexcess.com), we detect that this is a new user (based on having a valid auth session with clerk, but not having a user record in the internal database). We'll render any necessary "onboarding" views for the user, and in the background, the client will call the API to create a new user record.

## Authorization

TBD
