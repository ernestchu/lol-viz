# League of Legends Visualization

## Project Setup

```sh
yarn
```

### Setup for Riot API

Sign up and enroll in [Riot developer program](https://developer.riotgames.com/). Generate your own private API key and save it to `.env.local` in the project's root directory.
```sh
echo VITE_RIOT_API_KEY=(paste your API key here) > .env.local
```

### Compile and Hot-Reload for Development

```sh
yarn dev
```

### Compile and Minify for Production

```sh
yarn build
```

### Lint with [ESLint](https://eslint.org/)

```sh
yarn lint
```
