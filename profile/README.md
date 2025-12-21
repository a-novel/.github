<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/agora%20(dark).png">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/agora%20(light).png">
  <img alt="Agora logo." src="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/agora%20(dark).png">
</picture>

# ⌨️ We are Agora

Agora is an online writing studio, powered by AI. It helps you speed your writing process, and focus on creative tasks.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/studio%20(dark).png">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/studio%20(light).png">
  <img alt="Agora logo." src="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/agora%20(dark).png">
</picture>

Studio is our advanced creator tool for crafting engaging stories.

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/storyverse%20(dark).png">
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/storyverse%20(light).png">
  <img alt="Agora logo." src="https://raw.githubusercontent.com/a-novel/uikit/master/src/lib/assets/logos/HD/agora%20(dark).png">
</picture>

Storyverse is the upcoming consumer-focused platform, where you can enjoy your favorite stories, engage with friends and
creators, and way more!

## 📱 Socials

Join us on social media!

[![X (formerly Twitter) Follow](https://img.shields.io/twitter/follow/agorastoryverse)](https://twitter.com/agorastoryverse)
[![Discord](https://img.shields.io/discord/1315240114691248138?logo=discord)](https://discord.gg/rp4Qr8cA)
[![Spotify](https://img.shields.io/badge/Spotify-1ED760?logo=spotify&logoColor=white)](https://open.spotify.com/show/6NiTIuFh4vtNVoJzLozrWb)

# Technical

## Ports cheat sheet

We use the following system for selecting ports in development:

- `40--.70--` -> Indicates the category of the service using the port where
  - `---0,---9` indicates the sub-service (or dependency)

<table>

<tr><td>

| Category | Description           |
|----------|-----------------------|
| `40--`   | Backend service       |
| `50--`   | Backend test service  |
| `60--`   | Frontend service      |
| `70--`   | Frontend test service |

</td><td>

| Dependency / Sub-service | Description                 |
|--------------------------|-----------------------------|
| `---0`                   | Client application (client) |
| `---1`                   | Rest API           (rest)   |
| `---2`                   | GRPC Service       (grpc)   |
| `---3`                   | Postgres Database  (pg)     |
| `---4`                   | Mail server        (smtp)   |
| `---5`                   | Tolgee server      (tolgee) |

</td></tr>

</table>

| Service                                                                       | Port range     | Sub-services                                                |
|-------------------------------------------------------------------------------|----------------|-------------------------------------------------------------|
| [Service Json-Keys](https://github.com/a-novel/service-json-keys)             | `400-`, `500-` | grpc `4002, 5002`<br/>pg `4003, 5003`                       |
| [Service Authentication](https://github.com/a-novel/service-authentication)   | `401-`, `501-` | rest `4011, 5011`<br/>pg `4013, 5013`<br/>smtp `4014, 5014` |
| [Platform Authentication](https://github.com/a-novel/platform-authentication) | `600-`, `700-` | client `6000, 7000`<br/>tolgee `6005`                       |
| [UiKit](https://github.com/a-novel/uikit)                                     | `601-`, `701-` | tolgee `6015`                                               |
