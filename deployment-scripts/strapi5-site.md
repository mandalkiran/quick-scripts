---
title: Deploy Strapi5 Sites
description: This script will copy the project from repo, links environment, build app, activates deployment
tags: deployment, strapi, strapi5, sites
---
```
echo "Copy Project"
echo 'Cloning {{ repository }}:{{ branch }} to {{ releasePath }}'
mkdir -p {{ releasePath }}

echo 'Linking {{ latestPath }} to {{ releasePath }}'
rm -f {{ latestPath }} > /dev/null 2>&1
ln -sfn {{ releasePath }} {{ latestPath }} > /dev/null 2>&1

chmod 775 {{ projectPath }}/releases
{{ cloneCommand }} {{ releasePath }} --quiet

if [ -d {{ currentPath }} ]; then
  # if it's not a link, then we'll just back it up
  if [ ! -L {{ currentPath }} ]; then
    echo "Found an existing {{ currentPath }} directory. Removing it."
    rm -rf {{ currentPath }} > /dev/null 2>&1
  fi
fi
echo "Copy Project"
echo 'Cloning {{ repository }}:{{ branch }} to {{ releasePath }}'
mkdir -p {{ releasePath }}

echo 'Linking {{ latestPath }} to {{ releasePath }}'
rm -f {{ latestPath }} > /dev/null 2>&1
ln -sfn {{ releasePath }} {{ latestPath }} > /dev/null 2>&1

chmod 775 {{ projectPath }}/releases
{{ cloneCommand }} {{ releasePath }} --quiet

if [ -d {{ currentPath }} ]; then
  # if it's not a link, then we'll just back it up
  if [ ! -L {{ currentPath }} ]; then
    echo "Found an existing {{ currentPath }} directory. Removing it."
    rm -rf {{ currentPath }} > /dev/null 2>&1
  fi
fi

chmod 0600 /home/{{username}}/.ssh/id_rsa > /dev/null 2>&1


echo "Link ENV"

if [[ ! -f "{{ releasePath }}"/.env && -f "{{ projectPath }}/.env" ]]; then
  chown cleavr:cleavr {{ projectPath }}/.env
  chmod 0600 {{ projectPath }}/.env
  ln -s "{{ projectPath }}/.env" "{{ releasePath }}/.env"
  echo "Successfully linked {{ releasePath }}/.env to {{ projectPath }}/.env"
else
  echo "Env path {{ releasePath }}/.env is already linked to {{ projectPath }}/.env"
fi

echo "Install NPM Packages"

cd "{{ appFolderPath }}"
if [ -f yarn.lock ]; then
  echo 'Found yarn.lock. Installing packages yarn with frozen lockfile'
  yarn install --frozen-lockfile
elif [ -f package-lock.json ]; then
  echo 'Found package-lock.json. Installing packages using npm ci'
  npm ci
elif [ -f bun.lockb ]; then
  echo 'Found bun.lockb. Installing packages using bun install'
  bun --bun install
else
  echo 'Missing lock file(s). Installing packages using npm install'
  npm install
fi

echo "Build Assets"

if [[ ! -f "{{ releasePath }}"/.env && -f "{{ projectPath }}/.env" ]]; then
  chown cleavr:cleavr {{ projectPath }}/.env
  chmod 0600 {{ projectPath }}/.env
  ln -s "{{ projectPath }}/.env" "{{ releasePath }}/.env"
  echo "Successfully linked {{ releasePath }}/.env to {{ projectPath }}/.env"
else
  echo "Env path {{ releasePath }}/.env is already linked to {{ projectPath }}/.env"
fi

cd "{{ appFolderPath }}"
echo "Building the assets using npm run build --production ..."
NITRO_PRESET=cleavr npm run build --production


echo "Activate"


ASSETS_PATH="{{ fileUploadPath }}"
ORG_PATH="{{ appFolderPath }}/$ASSETS_PATH"

STORAGE_PATH="{{ projectPath }}/storage"
CANONICAL_PATH="$STORAGE_PATH/$ASSETS_PATH"
mkdir -p "$CANONICAL_PATH"

if [ -d "$ORG_PATH" ]; then
    echo "Found existing assets directory. Syncing its contents to persist the assets across deployments..."
    rsync -a "$ORG_PATH/" "$CANONICAL_PATH"
fi

echo "Replacing existing assets directory with a link to $CANONICAL_PATH"
rm -rf "$ORG_PATH"
ln -sf $CANONICAL_PATH $ORG_PATH

OLD_UPLOADS_DIR="{{ projectPath }}/uploads"
if [ -d "$OLD_UPLOADS_DIR" ]; then
    echo "Found old assets directory. Moving the assets so that they are synced up..."
    rsync -a --ignore-existing "$OLD_UPLOADS_DIR/" "$CANONICAL_PATH"
    rm -rf "$OLD_UPLOADS_DIR"
fi

echo "Setting correct permissions of the assets directory..."
chown -R {{ username }}:{{ username }} "$CANONICAL_PATH" || true
chmod -R 775 "$CANONICAL_PATH" || true



ln -sfn "{{ appFolderPath }}" "{{ currentPath }}"



if [ ! -f {{ appFolderPath }}/.cleavr.runner.js ]; then
  if [ ! -f {{ projectPath }}/.cleavr.runner.js ]; then
    echo "Strapi runner file {{ appFolderPath }}/.cleavr.runner.js doesn't exist. Creating one in {{ projectPath }}/.cleavr.runner.js..."
cat > {{ projectPath }}/.cleavr.runner.js << 'EOF'
const strapi = require('@strapi/strapi');
strapi.createStrapi({ distDir: "./dist" }).start();
EOF
  fi

  echo "Linking Strapi Runner..."
  cp {{ projectPath }}/.cleavr.runner.js {{ appFolderPath }}/.cleavr.runner.js
else
  echo "Strapi runner file found in {{ appFolderPath }}/.cleavr.runner.js. Linking Strapi runner..."
  cp {{ appFolderPath }}/.cleavr.runner.js {{ projectPath }}/.cleavr.runner.js
fi




if [ -f /home/{{ sudoUsername }}/.pm2/logs/{{ domain }}-out.log ]; then
  echo "Restarting {{ domain }}..."
  sudo /usr/sbin/service {{ domain }} restart
else

if [ ! -e {{ currentPath }}/.cleavr.config.js ]; then
echo "PM2 ecosystem file is not linked. Linking..."
ln -s {{ projectPath }}/.cleavr.config.js {{ currentPath }}/.cleavr.config.js
fi

  if [ -f {{ projectPath }}/.pm2_has_changed_since ]; then
    echo "PM2 ecosystem config has changed since the last deployment. Restarting the app..."
    echo "Stopping the old running app if any..."
    env PM2_HOME=/opt/pm2 pm2 del {{ domain }} || true
    echo "Starting the app..."
    env PM2_HOME=/opt/pm2 pm2 start {{ currentPath }}/.cleavr.config.js --update-env
  else
    echo "PM2 ecosystem config hasn't changed since the last deployment. Reloading the app..."
    env PM2_HOME=/opt/pm2 pm2 startOrReload {{ currentPath }}/.cleavr.config.js --update-env
  fi
  rm -f {{ projectPath }}/.pm2_has_changed_since > /dev/null 2>&1
# Restart if 502
status=`curl -I -s -L {{ domain }} 2>/dev/null | head -n 1 | cut -d$' ' -f2`
if [[ $status = "301" ]]; then
status=`curl  -s -o /dev/null -w "%{http_code}" -L {{ domain }} | head -n 1 | cut -d$' ' -f2`
fi
if [[ $status = "502" ]]; then
  echo 'Force restarting the app'
  env PM2_HOME=/opt/pm2 pm2 delete {{ domain }}
  env PM2_HOME=/opt/pm2 pm2 start {{ currentPath }}/.cleavr.config.js
fi

fi

chmod 0600 /home/{{username}}/.ssh/id_rsa > /dev/null 2>&1


echo "Link ENV"

if [[ ! -f "{{ releasePath }}"/.env && -f "{{ projectPath }}/.env" ]]; then
  chown cleavr:cleavr {{ projectPath }}/.env
  chmod 0600 {{ projectPath }}/.env
  ln -s "{{ projectPath }}/.env" "{{ releasePath }}/.env"
  echo "Successfully linked {{ releasePath }}/.env to {{ projectPath }}/.env"
else
  echo "Env path {{ releasePath }}/.env is already linked to {{ projectPath }}/.env"
fi

echo "Install NPM Packages"

cd "{{ appFolderPath }}"
if [ -f yarn.lock ]; then
  echo 'Found yarn.lock. Installing packages yarn with frozen lockfile'
  yarn install --frozen-lockfile
elif [ -f package-lock.json ]; then
  echo 'Found package-lock.json. Installing packages using npm ci'
  npm ci
elif [ -f bun.lockb ]; then
  echo 'Found bun.lockb. Installing packages using bun install'
  bun --bun install
else
  echo 'Missing lock file(s). Installing packages using npm install'
  npm install
fi

echo "Build Assets"

if [[ ! -f "{{ releasePath }}"/.env && -f "{{ projectPath }}/.env" ]]; then
  chown cleavr:cleavr {{ projectPath }}/.env
  chmod 0600 {{ projectPath }}/.env
  ln -s "{{ projectPath }}/.env" "{{ releasePath }}/.env"
  echo "Successfully linked {{ releasePath }}/.env to {{ projectPath }}/.env"
else
  echo "Env path {{ releasePath }}/.env is already linked to {{ projectPath }}/.env"
fi

cd "{{ appFolderPath }}"
echo "Building the assets using npm run build --production ..."
NITRO_PRESET=cleavr npm run build --production


echo "Activate"


ASSETS_PATH="{{ fileUploadPath }}"
ORG_PATH="{{ appFolderPath }}/$ASSETS_PATH"

STORAGE_PATH="{{ projectPath }}/storage"
CANONICAL_PATH="$STORAGE_PATH/$ASSETS_PATH"
mkdir -p "$CANONICAL_PATH"

if [ -d "$ORG_PATH" ]; then
    echo "Found existing assets directory. Syncing its contents to persist the assets across deployments..."
    rsync -a "$ORG_PATH/" "$CANONICAL_PATH"
fi

echo "Replacing existing assets directory with a link to $CANONICAL_PATH"
rm -rf "$ORG_PATH"
ln -sf $CANONICAL_PATH $ORG_PATH

OLD_UPLOADS_DIR="{{ projectPath }}/uploads"
if [ -d "$OLD_UPLOADS_DIR" ]; then
    echo "Found old assets directory. Moving the assets so that they are synced up..."
    rsync -a --ignore-existing "$OLD_UPLOADS_DIR/" "$CANONICAL_PATH"
    rm -rf "$OLD_UPLOADS_DIR"
fi

echo "Setting correct permissions of the assets directory..."
chown -R {{ username }}:{{ username }} "$CANONICAL_PATH" || true
chmod -R 775 "$CANONICAL_PATH" || true



ln -sfn "{{ appFolderPath }}" "{{ currentPath }}"



if [ ! -f {{ appFolderPath }}/.cleavr.runner.js ]; then
  if [ ! -f {{ projectPath }}/.cleavr.runner.js ]; then
    echo "Strapi runner file {{ appFolderPath }}/.cleavr.runner.js doesn't exist. Creating one in {{ projectPath }}/.cleavr.runner.js..."
cat > {{ projectPath }}/.cleavr.runner.js << 'EOF'
const strapi = require('@strapi/strapi');
strapi.createStrapi({ distDir: "./dist" }).start();
EOF
  fi

  echo "Linking Strapi Runner..."
  cp {{ projectPath }}/.cleavr.runner.js {{ appFolderPath }}/.cleavr.runner.js
else
  echo "Strapi runner file found in {{ appFolderPath }}/.cleavr.runner.js. Linking Strapi runner..."
  cp {{ appFolderPath }}/.cleavr.runner.js {{ projectPath }}/.cleavr.runner.js
fi




if [ -f /home/{{ sudoUsername }}/.pm2/logs/{{ domain }}-out.log ]; then
  echo "Restarting {{ domain }}..."
  sudo /usr/sbin/service {{ domain }} restart
else

if [ ! -e {{ currentPath }}/.cleavr.config.js ]; then
echo "PM2 ecosystem file is not linked. Linking..."
ln -s {{ projectPath }}/.cleavr.config.js {{ currentPath }}/.cleavr.config.js
fi

  if [ -f {{ projectPath }}/.pm2_has_changed_since ]; then
    echo "PM2 ecosystem config has changed since the last deployment. Restarting the app..."
    echo "Stopping the old running app if any..."
    env PM2_HOME=/opt/pm2 pm2 del {{ domain }} || true
    echo "Starting the app..."
    env PM2_HOME=/opt/pm2 pm2 start {{ currentPath }}/.cleavr.config.js --update-env
  else
    echo "PM2 ecosystem config hasn't changed since the last deployment. Reloading the app..."
    env PM2_HOME=/opt/pm2 pm2 startOrReload {{ currentPath }}/.cleavr.config.js --update-env
  fi
  rm -f {{ projectPath }}/.pm2_has_changed_since > /dev/null 2>&1
# Restart if 502
status=`curl -I -s -L {{ domain }} 2>/dev/null | head -n 1 | cut -d$' ' -f2`
if [[ $status = "301" ]]; then
status=`curl  -s -o /dev/null -w "%{http_code}" -L {{ domain }} | head -n 1 | cut -d$' ' -f2`
fi
if [[ $status = "502" ]]; then
  echo 'Force restarting the app'
  env PM2_HOME=/opt/pm2 pm2 delete {{ domain }}
  env PM2_HOME=/opt/pm2 pm2 start {{ currentPath }}/.cleavr.config.js
fi

fi
```
