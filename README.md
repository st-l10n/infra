# infra

## Pontoon server

Self-hosted instance of [Mozilla Pontoon](https://github.com/mozilla/pontoon).

Basically it is UI for collaborative translation implemented as Django application.

Requirements:

* PostgreSQL 
* RabbitMQ
* Celery worker that runs asynchronous jobs

## Periodical jobs

### Pontoon sync

Runs "sync_projects" job every 5 minutes.

It commits any string changes in the database to the remote VCS servers
associated with each project, and pulls down the latest changes to keep
the database in sync.

```console
$ python manage.py sync_projects
```

### Game sync

Runs "update-game.sh" script every 5 minutes.

Uses [SteamCMD](https://developer.valvesoftware.com/wiki/SteamCMD) (Steam console client) to periodically 
check for Stationeer updates (on "beta" branch) via special account.

If update is detected, script calls martian "update" command and commits changes.

Snippet from script:

```bash
echo "Martian start"
martian update --input ~/game/rocketstation_Data/StreamingAssets/
echo "Martian end"

find . -name "english*.xml" -type f -print0 | xargs -0 dos2unix
updateversion=$(grep -Po 'UPDATEVERSION=Update \S+' ~/game/rocketstation_Data/StreamingAssets/version.ini | sed -e "s/^UPDATEVERSION=Update //")
echo ${updateversion} > version.txt
find . -type f -name "english*.xml" -exec sha256sum "{}" \; > hash.txt

echo -e "Update ${updateversion}"
if [[ `git status --porcelain` ]]; then
  git config user.email "stationeers@gortc.io"
  git config user.name "Stationeers Bot"
  git add version.txt hash.txt
  git ls-files . | grep '\.xml$' | grep english --null | tr '\n' '\0' | xargs -0 -n1 git add
  git ls-files --others . | grep '\.xml$' | grep english --null | tr '\n' '\0' | xargs -0 -n1 git add
  git commit -m "automated update to ${updateversion}"
  git push origin master
else
  echo -e "No changes detected"
fi
```

Note that we can't use anonymous access here and login with password is required.
Moreover, every unique IP should be authenticated by Steam Guard manually.
Like that:
```console
$ ~/steam/steamcmd.sh
$ login stationeers_dl
Logging in user 'stationeers_dl' to Steam Public ...

password: 
$ ...
Steam Guard code:
$ 12345
```

## Triggered jobs

Currently, triggered jobs are managed by self-hosted [TeamCity](https://tc.st.gortc.io/) instance.

### Resources build

Triggered on update to [locales](https://github.com/st-l10n/locales) repo, which is managed by Pontoon and contains
`.po` files.

Clones [resources](https://github.com/st-l10n/resources) repo (content of StreamingAssets)
and runs `martian bake` command. This effectively performs translation from English resources, generating
translated resources from `english_*.xml` files using `.po` files from [locales](https://github.com/st-l10n/locales) as
translation source.

When translation is done, it commits generated files to [resources](https://github.com/st-l10n/resources) repo,
generates `StreamingAssets.zip` file that is pushed to discord channel via `martian notify` command:
```console
martian notify --token ${discord_token} assets
```

### Locales build

Triggered on update to [resources](https://github.com/st-l10n/resources) that touches `english_*.xml` files
(the exact trigger is `hash.txt` file).

Runs `martian generate` commands:
```bash
martian generate --output locales-new
martian generate --limit en --output locales-new --template=false
```

So it uses [resources](https://github.com/st-l10n/resources) repo to produce update to [locales](https://github.com/st-l10n/locales) repo.

This will generate `.pot` files that will be loaded to Pontoon, making them available for translation.
