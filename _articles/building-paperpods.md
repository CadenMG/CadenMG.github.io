---
id: 4
title: "Building PaperPods"
subtitle: "A tour of modern web app development"
date: "2024.04.13"
tags: "product, paperpods"
---

![paperpods](/images/paperpods.svg)

### PaperPods
PaperPods is as *standard* and unremarkable of a web-app as any. There's a landing page, authentication, authorization, payments, data storage, data retrieval, and some backend.
You go to the site, signup, pay for the service, use the service, and expect your data to persist. That's about it.
But while the end-result is honestly pretty trivial, you're often faced with many different options for each piece of the website. I'll highlight some of the decisions I made along the way, and perhaps you can use this as an n=1 example for why to use some framework.

### Cloud Provider
At the heart of most modern web development is some cloud infrastructure. I've become a fan of GCP in recent years - mostly because it's the one I have the most experience with - but I really just appreciate how dead simple [Firebase](https://firebase.google.com/) is to setup and get going ASAP.
Obviously GCP isn't the only solution. But I'm not here to give you all of the solutions, just the ones I've found to like 😊

### Authentication
Firebase. Create an account, smash a few buttons, copy some template, and you've got a secure, working signup + login page.
I'm sure there exists other solutions which are equally as simple, but I can't see it getting any easier than this.

### Data Storage
Firebase offers a few different storage options. For key/value based storage, I've used Firestore Database and haven't regretted it yet. I also use Firestore Storage for my blob/file based storage. I don't think Firestore has any relation DB solutions currently, but GCP's Cloud SQL should have you covered.

### Serverless Functions
Docker with GCP's Cloud Run. Build it, upload to GCR, and deploy to Cloud Run. Simple. And there's plenty of customization's you can do around the machine specs and runtime conditions, if you need. I use it for running all of my Docker containers.

### Serverless Jobs
I'm defining this as code which executes async. in response to some event. Firebase has Cloud Functions which allows you to do just that. I use it for sending out my weekly newsletters.

### Continuous Deployment
Who doesn't love GitHub workflows 😊? I jest, but it does do a good job of building Docker containers and deploying them to Cloud Run for me.

### Hosting
[Vercel](https://vercel.com/). I've heard mixed things on all of the major ones, and I'm not going to pretend to be an expert, but it seems decent so far. I like that you can trigger deployments on pushes to certain branches. And it handles environment variables in a clean way.

### Domains
[Namecheap](https://www.namecheap.com/). Not much to say about this one. Go buy a domain for cheap.

### Forms
Forms are a nice way for user's to contact you about issues or questions. [Formspree](https://formspree.io/) was easy to integrate and gets the job done.

### Email Marketing
I have mixed feelings on this one, but I've used [Mailchimp](https://mailchimp.com/) and have had mixed success. It's a powerful tool, and I'm sure it's got its use-cases, but honestly if you're just trying to do something simple (like I was...), then maybe look elsewhere.

### Payments
[Stripe](https://stripe.com/). I guess it's the gold standard at this point. I'd say it deserves it. Fairly easy to integrate, test in dev environments, and setup in production.
