# Environment Configuration

To make this pipeline lightning fast and connect it to your streaming platforms, you need to configure your environment variables.

## 1. Streaming Targets (`scripts/streaming/rtmp_targets.env`)

This file is ignored by git for your safety. It stores your private stream keys.
Create it by copying the template:
```bash
cp scripts/streaming/targets.env.template scripts/streaming/rtmp_targets.env
```

Open `rtmp_targets.env` and add your keys:
```bash
TWITCH_URL="rtmp://live.twitch.tv/app/live_XXX_YYY"
YOUTUBE_URL="rtmp://a.rtmp.youtube.com/live2/XXXX-YYYY-ZZZZ"
# FACEBOOK_URL="rtmps://live-api-s.facebook.com:443/rtmp/XXX"
```

## 2. Hugging Face Token (`HF_TOKEN`)

Model downloads can be slow or rate-limited if you pull anonymously. Set your Hugging Face token to speed up downloads and ensure reliable model access.

Add it to your environment or a generic `.env` file at the root:
```bash
export HF_TOKEN="hf_your_token_here"
```

For Cloud Run / GPU instances, this token will be injected via Google Secret Manager to automatically pull models dynamically without baking them into the container image on every startup.
