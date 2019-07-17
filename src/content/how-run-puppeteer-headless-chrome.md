---
draft: false
title: 'How to run Puppeteer using Cloud Functions'
author: jmolivas
excerpt: lorem ipsum bar/baz
image: firebase-cloud-function-puppeteer.jpeg
date: '2019-01-'
path: ''
tags: []
updated_at: '2019-07-17T05:55:08.855Z'
---

After following a discussion at the GatsbyJS discord space about running puppeter using a cloud function. I decided to give it a try and in this post I will share my experience.

### What is Puppeteer?

Puppeteer is a Node library which provides a high-level API to control Chrome or Chromium over the DevTools Protocol. Puppeteer runs headless by default.

### Why Puppeteer?

This library comes with everything you need to:

* Generate screenshots and PDFs of pages.
* Crawl a SPA.
* Pre-render content (i.e. "SSR" (Server-Side Rendering)).
* Automate form submission, UI testing, keyboard input, etc.

For te purpose of tihs example we are going to take advantage of the screenshot generation.

### Which Cloud Provider?

At [weKnow](https://weknowinc.com/) we have been using and having a great experience with Google Cloud Services and for the purpose of this example. I will be using a Cloud Function from Google Firebase service

### Adding the puppeteer dependency

To use Puppeteer in your project using `npm` execute on your functions directory:

```
npm install --save puppeteer

```

### The Firebase Cloud Function

The following example creates a new `http` function to handle the request using the `<https://path-to-your-project.web.app>/screeshot` url. This returns a screenshot of the provided `url` query parameter.

``` javascript
exports.screenshot = functions
  .runWith({
    timeoutSeconds: 120,
    memory: "2GB"
  })
  .https.onRequest((request, response) => {
    (async () => {
      const url = request.query.url;
      const width = parseInt(request.query.width) || 1024;
      const height = parseInt(request.query.height) || 768;
      const fullPage = request.query.full
        ? request.query.full === "true"
        : false;

      if (!url) {
        return response.send(`Invalid url: ${url}`);
      }

      const browser = await puppeteer.launch({
        args: ["--no-sandbox"]
      });
      const page = await browser.newPage();

      await page.goto(url, { waitUntil: "networkidle2" });

      if (!fullPage) {
        await page.setViewport({ width, height });
      }

      const screenshot = await page.screenshot({ fullPage });

      await browser.close();

      return response.type("image/png").send(screenshot);
    })();
  });

```

> Note the **runWith** method is used in order to set timeout and memory allocation used on the cloud function execution.
