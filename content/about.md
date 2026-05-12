---
title: "About"
layout: "about"
# url: "/about"
summary: "about"
---
# About Me

### The Intersection of Design and Engineering

My journey in the digital space began with a focus on aesthetics. As a former designer, I learned how users perceive and interact with interfaces. However, my "curious geek" heart eventually pulled me deeper into the machine. Since 2018, I have been building the web professionally, evolving into a **App Developer** able to implement small scale devops.

While I spend my days building modern web applications, my nights are spent in the "engine room" — a home lab that serves as my personal playground for innovation, automation, and infrastructure testing. 

### What You’ll Find Here

This blog is a living record of my experiments and the inevitable "tinkering" that comes with being a lifelong learner. I document my process to help others navigating the intersection of web development and self-hosted infrastructure.

**I write about:**

- **Web Engineering:** Frontend frameworks (Angular/TypeScript), Unit Testing, and performance optimization.
    
- **Infrastructure & DevOps:** Docker Compose, Proxmox virtualization, and CI/CD with Dokploy.
    
- **System Administration:** Hardening networks, Linux command-line mastery, and Python automation for home sensors.
    
- **The Designer’s Perspective:** Why UI/UX and visual clarity still matter when building high-performance technical tools.

### My Technical Playground (The Home Lab)

I believe in "learning by doing." To truly understand how the web works, I maintain a local environment where I can break things and rebuild them better:

- **The Testing Ground:** A **Mac Mini** dedicated to prototyping containerized applications and testing new frameworks.
    
- **The Server:** A **Minisforum** running **Proxmox**. This is where I manage LXC and Docker containers using **Dokploy**.
    
- **The Network:** A hardened environment secured by an **OPNsense** firewall on a dedicated Dell tower, using **Cloudflared** and **Tailscale** for secure remote access.

```markdown
   [ Public Internet ]
            |
 [ Cloudflare/Tailscale ]
            |
            v
    +----------------+
    |  OPNsense FW   | 
    +-------+--------+
            |
            |
     +-------------+        +-----------+
     | Minisforum  |        | Mac Mini  | 
     | (Proxmox)   |------->| (Testing) | 
     +------+------+        +-----+-----+
            |                     |       
            |               [ Local Dev ] 
     +------+------+
     |             |
 [ Docker ]    [  LXC  ]
 [ Dokploy]        |
     |             |
 [  Prod  ]    [ Sensors ]
```   

### Let’s Connect

Thanks for stopping by my corner of the web. Whether you’re here for a specific technical fix, curious about home lab setups, or just browsing, I appreciate the visit. This site is a perpetual work-in-progress—much like my servers—and I’m always happy to discuss web development, infrastructure, or the latest in local AI.

If you’d like to keep in touch or discuss a project, you can find me here:

- **LinkedIn:** [linkedin.com/in/liviuiancu](https://www.linkedin.com/in/liviuiancu/) — For professional inquiries and networking.
    
- **GitHub:** [github.com/liviuiancu](https://www.google.com/search?q=https://github.com/your-username) — Where I push my experiments and configuration snippets.

Stay curious!
