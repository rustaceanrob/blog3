---
title: "First Thoughts on Astro"
date: "Jan 17 2024"
description: "My first day using a project with lots of buzz"

---

## My Blog Struggles

I wanted to upgrade my portfolio and add a little personality to my website, 
so I tried to include a blog in my bare React and Tailwind app. My first take was to use my existing Firebase project, upload markdown files to a storage bucket, and pull them into my React components. I got through some of this process, but I got extremely bored. Time to find some lower hanging fruit.

## Enter Astro

At the core, all I want to do was bundle my markdown along with my site and make updates to the site by adding new markdowns to the project itself. In retrospect, the solution was very obviously Astro, and they even had a starter template for a blog when using `npm create astro`. For mostly static apps, I instantly knew I found my solution as long as Astro is maintained. 

## Ecosystem

My first impression was very good, but it only got better. The **Themes** community is insane. This website is forked from an Astro theme, but they have much more than blogs. If I am making a landing page, documentation, or design portfolio, there is no question with what I'm going with. 

## The Competition

Where I will probably fall short of using Astro is for apps with lots of user interactions, webhooks, third-party integrations, etc. I know the Astro team is working to make these features seemless, but NextJS feels all-to-familiar after coming from bare React. At the end of the day, I am going to go with whatever is easiest to bootstrap.

