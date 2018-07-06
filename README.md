## Make Travis-CI deploy to S3 and notify on Slack

This makes all code commits automatically available for testing at minimum cost.

### Prerequisites

To make use of this, you'll need

* [Travis-CI account](https://travis-ci.com/)
* [Github account](https://github.com/)
* a codebase
* [Slack.com](https://slack.com/) account with admin access (or someone who can create an incoming-webhooks)
* [AWS Account](https://console.aws.amazon.com/) with admin access (or someone who can issue IAM user with S3 access)

### Preperation
* Enable your repository with Travis CI.
* In Travis CI repository settings (https://travis-ci.com/[organisation]/[repo]/settings) create the following environmental variables
```
AWS_ACCESS_KEY_ID - IAM user with write access to S3
AWS_SECRET_ACCESS_KEY - secret part of the above
AWS_REGION - AWS region for your s3 buckets (i.e. "eu-west-1")
SLACK_NOTIFICATION_URL - incoming-webhooks URL created under slack integrations settings 
SLACK_USERNAME - username used for slack notifications
SLACK_CHANNEL - slack channel used notifications (starting with #) - isolated channel is recommended
```

### Let's do it

In .travis.yml install aws dependencies at before_script, it needed to deal with s3 buckets
```yml
before_script:
  # installs aws tools needed to deal with s3 buckets
  - gem update --system
  - sudo apt-get install -y python3.4
  - sudo apt-get install --upgrade -y python-pip
  - sudo pip install --upgrade awscli
  # creates branch link - you may want to change 'testing-' to something more related to what you do
  - "export S3_DEPLOY_BUCKET=testing-${TRAVIS_BRANCH//_/-}"
  - "export APP_WEBHOST=http://$S3_DEPLOY_BUCKET.s3-website-$AWS_REGION.amazonaws.com"
```

At `script` stage

```yml
script:
  # creates s3 bucket if doesnt exist
  - |
    if aws s3api head-bucket --bucket "$S3_DEPLOY_BUCKET" 2>/dev/null; then 
      echo "bucket $S3_DEPLOY_BUCKET already exists"
    else
      aws s3 mb s3://$S3_DEPLOY_BUCKET --region $AWS_REGION
      aws s3 website s3://$S3_DEPLOY_BUCKET --index-document index.html --error-document index.html  
    fi
# run your tests & build here
  - npm install
  - npm run test
  - npm build

```

Let's assume your `npm build` process generated a build in /dist folder, this may be different for your project

At `deploy` stage we use Travis deployment tools to push our build to s3
```yml
deploy:
  - provider: s3
    access_key_id: $AWS_ACCESS_KEY_ID
    secret_access_key: $AWS_SECRET_ACCESS_KEY
    region: $AWS_REGION
    bucket: $S3_DEPLOY_BUCKET
    skip_cleanup: true
    local_dir: dist
    acl: public_read
    on:
      all_branches: true
```

At `after_deploy` stage we export some meta data about latest code change and fire notification
```yml
after_deploy:
  # sends slack notification with test link, branch name, author and last commit message
  - export AUTHOR_NAME="$(git log -1 $TRAVIS_COMMIT --pretty="%aN")"
  - export LAST_COMMIT_COMMENT="$(git log -1 --pretty="%B")"
  - "curl -X POST --data-urlencode \"payload={\'channel\':\'SLACK_CHANNEL\',\'username\':\'$SLACK_USERNAME\',\'text\':\'*$AUTHOR_NAME* pushed *$TRAVIS_BRANCH* branch to :cloud: <$APP_WEBHOST| click here> to test, last message: _$LAST_COMMIT_COMMENT _\'}\" $SLACK_NOTIFICATION_URL --insecure"
```


### Thats it
Your CI just got new powers, enjoy!

### To-do
The above solution is not issue-free, there is few things to remember:
* AWS limits number of buckets to 100 by default, you may want to ask them for more
* This need a garbage collector mechanism to clean old, unused buckets
* It would be good to have a solution with single bucket and multiple folders
