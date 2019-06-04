Developer workflow with Jenkins
===============================

## Prerequisites

1: A git repository to store `Jenkinsfile` and api definitions

2: Jenkins installation

```
docker run -p 8082:8080 -p 50000:50000 \
-v jenkins_home:/var/jenkins_home \
jenkins/jenkins:lts-alpine
```

3: Tyk-Sync installed locally

`go install -u github.com/TykTechnologies/tyk-sync`

4: A local / remote development Tyk installation

We shall assume that the url for the local dashboard is `http://localdash:3000`

5: A production Tyk installation

We shall assume that the url for the prod dashboard is `http://proddashboard:3000`

---

## Setup

Copy and commit Jenkinsfile from this directory to the git repo in prerequisite 1

Configure a jenkins multi-branch pipeline which is able to connect to the git repository. 
You may need to enable auth for this if the repo is private.

Log into jenkins and add some credentials for the production installation which the Jenkins 
pipeline script should be able to access

```
TYK_DASH_URL = credentials('tyk-dash-url')
TYK_ORG_ID = credentials('tyk-org-id')
TYK_DASH_SECRET = credentials('tyk-dash-secret')
```

---

## Example Workflow

update local master in case any team mates have deployed to production

```
git checkout master && git remote update && git pull
```

publish possible changes to local dev environment

```
tyk-sync publish -d http://localdash:3000 -o LOCAL_ORG_ID -s LOCAL_SECRET git@github.com:foo/bar.git -b refs/heads/master
```

checkout to new branch

```
git checkout -b api/my-foo
```

design apis in local dev installation and dump your local dev env with your own changes to disk

```
tyk-sync dump -d http://localdash:3000 -s LOCAL_SECRET
```

commit to git and push

```
git add . && git commit -m "created my new api" && git push
```

---


## Unit testing

visit Jenkins this should trigger a build, if local jenkins installation, then tell it to build your branch

because you are not building the master branch, it will not deploy, only run unit tests

if the tests pass, then you should have a nice green box letting you know that everything is OK

if the tests fail, you should fix your api-definitions, rinse and repeat till it works

unit tests in this Jenkinsfile are very simple example written in Jenkins groovy declarative
syntax, but really you should write your own, depending on what you want to achieve:

```
def static assertAuthenticated(api) {
    assert api.use_keyless == false : "api ${api.name} should have authentication enabled"
}

def static assertWhitelistedTag(api) {
    assert api.tags.size() > 0 : "api ${api.name}  should be tagged either internal, external or both"

    api.tags.each { tag ->
        assert ["internal", "external"].contains(tag) : "api ${api.name} contains unknown tag ${tag}"
    }
}
```

the above assertion functions check that all apis have auth enabled, and that all apis are sharded with
either internal, external or both.

---

## Deploy to Production

so the tests pass - everything is green - woo hoo

open a pull request in github, merge to master, then go back to jenkins to trigger the build if not
automatic

because we are in master branch - jenkins will use tyk-sync to sync the master branch with production

