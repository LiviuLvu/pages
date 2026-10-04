---
title: "Building a small web app to control my Miele robot vacuum"
date: "2026-10-04"
summary: "How I connected a Miele Scout RX3 to a small application, deployed it in my home lab, and shared acces to family members through cloudflare tunnel."
# description: ""
tags: ["Vue", "Hono", "Docker", "Proxmox", "home lab", "API"]
author: "Liviu Iancu"
weight: 1
series: ["Programming"]
# categories: [""]
# aliases: [""]
# cover:
#   image: images/img.png
#   caption: "Description [text](https://link.somewhere/)"
---

I have a Miele Scout RX3 robot vacuum. It works well from the normal application, but I wanted to know how far I could go with the developer API and my home lab.

The result is a small application. It gives me a web dashboard for the vacuum, lets me start and stop cleaning, shows live status updates, and can start cleaning on a local schedule.

This started as a simple idea: make one API request and display the response. The first request was the easy part. Authentication, stale status, device capabilities, Server-Sent Events, safe controls and deployment were the parts that made the project interesting.

## The goal

I only wanted a small private tool with the controls and information that are useful to me:

- see whether the RX3 is cleaning or docked;
- see battery, signal and current program information;
- select Auto, Spot, Turbo or Silent mode;
- start, pause, stop, power on and power off the robot when the API allows it;
- receive changes without repeatedly refreshing the page;
- start cleaning on selected weekdays at a selected time.

I also wanted to keep the project inside my home lab instead of exposing a development server directly to the internet.

## The application layout

The application is split into a Vue frontend and a Hono backend. Docker Compose runs the services locally and a separate production Compose file runs the deployed version.

```text
Browser
  -> Cloudflare Tunnel
  -> Nginx web container
  -> Hono API container
  -> Miele API
```

The browser never talks directly to Miele. The API owns the OAuth flow, refresh token and device selection. The web container serves the built Vue application and proxies API requests to the private API container.

Production runs on a small Ubuntu VM in Proxmox. The VM has one CPU, 1 GB of RAM and a 20 GB disk. It is enough for this application because there is no database server or large monitoring stack. Docker runs the web, API and Cloudflare Tunnel containers.

The Cloudflare Tunnel can reach only the web container. The API is not published as a host port and is not connected to the Cloudflare network. This is a small detail, but it is a better boundary than putting every container on the public path.

## Finding the useful Miele fields

The developer API exposes more data than I needed. I inspected the device identification, state and available-actions endpoints first, then implemented only fields that I could verify on this RX3.

The status response includes the current program, cleaning state, elapsed time, battery, signal information, Mobile Start and Smart Grid values. Some fields looked useful but were not reliable in practice. For example, `remainingTime` and `startTime` were returned as `[0, 0]` placeholders, so I did not display them as if they contained real values.

The same approach applies to faults. The robot manual contains display codes such as F1 through F8, but the current API response does not expose a verified mapping to those codes. A boolean signal failure is not enough information to tell the user that a particular motor or brush has a problem. It is better to show less information than to invent a diagnosis.

The API also has an actions endpoint. This is important for controls because the UI should not assume that every action is available at every moment. Pause and Stop are shown only when the live capabilities advertise them. Power off is treated in the same way.

## OAuth and the first API call

The application uses Miele OAuth2. The backend starts the authorization request, stores a short-lived state value, and binds that state to the application session. After the callback, the refresh token is stored in a Docker-managed volume rather than in the source repository.

The first version was too optimistic about refresh tokens. Miele can rotate a refresh token, so two simultaneous requests cannot refresh independently and then both save their result. The backend now serializes refresh operations and lets concurrent requests share the same refresh promise.

This is one of those bugs that is easy to miss when testing with one browser request at a time. It becomes much more likely when the page loads status and actions together, or when several status refreshes happen after an SSE event.

## Making the dashboard live

Polling would have been simple, but it felt wasteful for a device that already provides Server-Sent Events. The API proxies the verified Miele event stream to the browser.

The browser does not treat every event payload as the complete truth. Miele can send an event saying that devices or actions changed while the details in the event are stale. Instead, an interesting event schedules an authoritative status request.

There is also a small amount of coordination around these requests. A burst of events is allowed to create one active status request and at most one follow-up request. Without that, a single update could create a queue of identical calls.

When the stream disconnects, the client reconnects and refreshes status after recovery. A warning is displayed while live updates are unavailable, but a temporary stream problem does not make Miele appear disconnected.

## Controls need more care than status

Reading a vacuum status is harmless. Starting or stopping it is not.

Every control is sent immediately and is disabled while the request is in flight. The backend checks the live capabilities again before making the Miele request instead of trusting an old browser response.

The API also requires a same-origin request for every state-changing route. A session cookie alone is not enough protection for a private application. The server checks the exact scheme, host and port from the `Origin` header. This covers mode changes, cleaning controls, power controls, sign-out and the OAuth start route.

The application has one configured account. There is no registration or user-management screen. Sessions are unique, stored in memory and revoked on sign-out. They expire after 48 hours, and the Miele event stream is closed when the application session is revoked or expires.

This is more work than adding a password field to a small dashboard, but the dashboard can start a machine in my home. It deserves the same basic care as any other control interface.

The main benefit of this is to simplify the access to control the robot for the entire family.

## A schedule without a provider schedule API

I first checked whether the Miele API could create and manage schedules. It did not provide a usable contract for this RX3, so I made the scheduler part instead of pretending that the provider supported it.

There is one account-wide schedule with a start time and selected weekdays. The browser sends its IANA timezone when saving. The server stores the schedule and recent occurrence results in a JSON file inside the application data volume.

The scheduler reads the current RX3 status immediately before starting. It uses the current mode rather than saving a mode inside the schedule. If the vacuum is already cleaning, the occurrence is recorded as skipped. If the app is down when the scheduled time passes, it is marked as missed instead of starting late after recovery.

The awkward cases are calendar cases. The scheduler works with calendar dates rather than just adding 24 hours to a timestamp. A nonexistent spring daylight-saving time is recorded as missed, and a repeated autumn time runs once. A pending occurrence is also resolved as unknown after a restart instead of being blindly retried.

This is more code than a simple `setInterval`, but a timer that works on my laptop is not the same thing as a schedule that is safe to run in a container.

## Monitoring the provider

An integration with a third-party API can silently stop working when the provider changes a response or an endpoint. I added a daily read-only monitor for that reason.

It checks device discovery, actions, identification, state and SSE negotiation. It validates the primitive types of fields consumed by the application, but it does not send a control request. Starting a cleaning cycle at midnight just to check an API would be a bad monitoring strategy.

The monitor writes only the check time, result category and failed endpoint names to a small status file. The production VM runs it from a systemd timer, and my homelab monitoring can alert if the check fails.

After login, the dashboard can show a dismissible warning when the latest result indicates an API contract change or unavailable endpoint. This is useful for discovering problems with showing the status and other readings. The endpoints that change the robot state are excluded from daily checks and only indicate a problem when the user can not trigger an action.

## What I learned

The most useful lesson was that the API request is only the beginning. A small integration still needs answers for stale data, concurrent refreshes, capability changes, failed writes, lost event streams, authentication expiry and restarts.

I also learned to avoid implementing fields just because they exist in a response. The API exposes values that are not meaningful for this device, and the robot manual contains information that the API does not expose. Verified limitations are part of the application design.

The home lab made it possible to experiment without another monthly cloud bill. Proxmox provides the VM, Docker keeps the deployment small, and the Cloudflare Tunnel avoids opening an inbound port. The tradeoff is that I have to manage the VM and the tunnel configuration.

## Next steps

The main unfinished feature is better RX3 activity and fault intelligence. I captured phases such as cleaning and returning to the base station, but I still need a real fault response before connecting the manual's F1 through F8 messages to the dashboard.

I also need browser tests for the scheduled cleaning controls. The API has tests for timing, recovery and duplicate prevention, but the visible schedule editing flow still needs end-to-end coverage.

For now, the app does what I wanted: it is a small private dashboard that controls one robot, gives me useful status, and makes a practical home lab project out of an appliance that normally stays behind a closed application.

If you found any useful lessons in this post and it provided inspiration for your projects, let me know.
