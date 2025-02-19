ARG NODE_VERSION=16
FROM node:${NODE_VERSION}-alpine

LABEL maintainer="support@apify.com" Description="Base image for Apify actors using headless Chrome"

# This image was inspired by https://github.com/GoogleChrome/puppeteer/blob/master/docs/troubleshooting.md#running-puppeteer-in-docker

# Disable chrome auto updates, based on https://support.google.com/chrome/a/answer/9052345
RUN mkdir -p /etc/default && echo 'repo_add_once=false' > /etc/default/google-chrome

# Install latest Chrome dev packages and fonts to support major charsets (Chinese, Japanese, Arabic, Hebrew, Thai and a few others)
# Note: this also installs the necessary libs to make the bundled version of Chromium that Puppeteer installs, work.
RUN DEBIAN_FRONTEND=noninteractive apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y wget gnupg unzip ca-certificates --no-install-recommends \
    && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | DEBIAN_FRONTEND=noninteractive apt-key add - \
    && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
    && DEBIAN_FRONTEND=noninteractive apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get purge --auto-remove -y wget unzip \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y procps git google-chrome-stable libxss1 fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst ttf-freefont \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /src/*.deb


# Add user so we don't need --no-sandbox.
RUN groupadd -r myuser && useradd -r -g myuser -G audio,video myuser \
    && mkdir -p /home/myuser/Downloads \
    && chown -R myuser:myuser /home/myuser

RUN mkdir -p /etc/opt/chrome/policies/managed \
    && echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' > /etc/opt/chrome/policies/managed/managed_policies.json \
    && echo '{ "ComponentUpdatesEnabled": "false" }' > /etc/opt/chrome/policies/managed/component_update.json

# Run everything after as non-privileged user.
USER myuser
WORKDIR /home/myuser

# Copy source code
COPY package.json main.js puppeteer_chrome_test.js /home/myuser/

# Uncomment to skip the chromium download when installing puppeteer. If you do,
# you'll need to launch puppeteer with:
#     browser.launch({executablePath: 'google-chrome'})
# ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD true

# Sets path to Chrome executable, this is used by Apify.launchPuppeteer()
ENV APIFY_CHROME_EXECUTABLE_PATH=/usr/bin/google-chrome

# Tell Node.js this is a production environemnt
ENV NODE_ENV=production

# Enable Node.js process to use a lot of memory (actor has limit of 32GB)
# Increases default size of headers. The original limit was 80kb, but from node 10+ they decided to lower it to 8kb.
# However they did not think about all the sites there with large headers,
# so we put back the old limit of 80kb, which seems to work just fine.
ENV NODE_OPTIONS="--max_old_space_size=30000 --max-http-header-size=80000"

# Install default dependencies, print versions of everything
RUN npm --quiet set progress=false \
    && npm config --global set update-notifier false \
    && npm install --only=prod --no-optional --no-package-lock --prefer-online \
    && echo "Installed NPM packages:" \
    && (npm list --only=prod --no-optional || true) \
    && echo "Node.js version:" \
    && node --version \
    && echo "NPM version:" \
    && npm --version \
    && echo "Google Chrome version:" \
    && bash -c "$APIFY_CHROME_EXECUTABLE_PATH --version"

# NOTEs:
# - This needs to be compatible with CLI.
# - Using CMD instead of ENTRYPOINT, to allow manual overriding
CMD npm start --silent
