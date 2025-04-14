# KMP Publish iOS App to TestFlight

This GitHub Action automates the process of building and distributing an iOS application to TestFlight using Fastlane. It supports secure code signing via Fastlane Match, handles build number incrementing, and uploads the app to TestFlight via App Store Connect API keys.

Start from local first.</strong> If it works locally, it will definitely work on CI.

<hr data-start="378" data-end="381" class="">

## 1. Local Setup (Required Once)

1. **Install Ruby**  
   [https://www.ruby-lang.org/en/documentation/installation/](https://www.ruby-lang.org/en/documentation/installation/)

2. **Install Fastlane**  
   [https://docs.fastlane.tools/getting-started/ios/setup/](https://docs.fastlane.tools/getting-started/ios/setup/)

```
ruby -v
fastlane -v
```
<hr data-start="378" data-end="381" class="">

## 2. Apple Developer Setup
- Go to [Apple Developer Console](https://developer.apple.com/account).

- Make sure you have at least one registered device:  
  https://developer.apple.com/account/resources/devices/list

<hr data-start="378" data-end="381" class="">

## 3. Set Up Fastlane in Your KMP iOS Project
In your terminal: (inside your root project)

```
fastlane init
```

When prompted:
1. Choose option:
   `2. Automate beta distribution to TestFlight`

2. Apple ID Email:
   Type your Apple Developer email (e.g., yourname@icloud.com).

3. App Identifier:
   If it doesn't exist, you'll see:
```
Checking if the app 'org.your.bundle.id' exists in your Apple Developer Portal..
It looks like the app 'org.your.bundle.id' isn't available on the Apple Developer Portal
Do you want fastlane to create the App ID for you on the Apple Developer Portal? (y/n)
```
Type y
Then enter App Name (can be anything, just make sure it's unique)

4. Then:
```
Checking if the app 'org.your.bundle.id' exists on App Store Connect..
It looks like the app isn't available on App Store Connect
Do you want fastlane to create the App on App Store Connect for you? (y/n)
```

You can type y, but this step sometimes fails.

If it fails, go manually:  https://appstoreconnect.apple.com/apps → + New App → Fill required fields
Then return to the terminal and re-run the setup.

<hr data-start="378" data-end="381" class="">

## 4. Generated Files
After init, you’ll see:
```
rootProject/
└── fastlane/
    ├── Appfile
    └── Fastfile
```
These contain basic metadata and CI/CD setup.

<hr data-start="378" data-end="381" class="">

## 5. Configure Code Signing with fastlane match
Now let’s set up automatic provisioning and code signing with Fastlane Match.

Step-by-step:
1. Run Match Init
```
fastlane match init
```

2. Choose Storage Type

It will ask:
<blockquote data-start="293" data-end="376">
<p data-start="295" data-end="376" class=""><strong>fastlane match</strong> supports multiple storage options. Please select the one you want to use:.</p>
</blockquote>

```
1. git
2. google_cloud
3. s3
4. gitlab_secure_files
```
Type 1 and press enter.

We’ll store certificates and provisioning profiles in a private Git repository.

3. Create a Private Git Repo

- Go to GitHub and create a new private repository (e.g., `your-app-match-certificates`).

- Copy the HTTPS URL (e.g., `https://github.com/your-org/your-app-match-certificates.git`).

- Paste it in the terminal when prompted:

<blockquote data-start="293" data-end="376">
<p data-start="295" data-end="376" class="">URL of the Git Repo:<br>
https://github.com/your-org/your-app-match-certificates.git.</p>
</blockquote>

4. Edit Matchfile

Open the newly created Matchfile in your iosApp/fastlane/ folder. You’ll see:
```
type("development")
```
Replace "development" with "appstore" since we’re deploying to TestFlight:
```
type("appstore")
```
<hr data-start="378" data-end="381" class="">

## 6. Generate Certificates & Profiles
Now we’ll actually create and fetch the necessary provisioning profile and certificate.
```
fastlane match appstore
```
It will ask:
```
Enter the passphrase that should be used to encrypt/decrypt your certificates
```
Create a secure passphrase (save it somewhere!). This will encrypt your certificates in the Git repo.

You’ll later add this to your GitHub repo as a secret:
```
MATCH_PASSWORD=your_passphrase
```

<hr data-start="378" data-end="381" class="">

## 7. Verify on Apple Developer Portal
After the above, Fastlane will automatically:

- Create a Distribution Certificate at [Certificates List](https://developer.apple.com/account/resources/certificates/list)

- Create a Provisioning Profile at [Profiles List](https://developer.apple.com/account/resources/profiles/list)

<hr data-start="378" data-end="381" class="">

## 8. Xcode Configuration
Now link the generated provisioning profile to your project manually:

1. Open iosApp.xcodeproj in Xcode.

2. Click your project in the navigator (top left).

3. Under Targets, click your app.

4. Go to the Signing & Capabilities tab.

5. Uncheck “Automatically manage signing”

6. From Provisioning Profile, select the one that was created via fastlane match.

<hr data-start="378" data-end="381" class="">

## 9. Create App Store Connect API Key
1. Go to [Users and Access](https://appstoreconnect.apple.com/access/api) → Keys

2. Click + to generate a new key

3. You'll get:

- Key ID

- Issuer ID

- Private key (.p8 file)

Save them somewhere secure.

<hr data-start="378" data-end="381" class="">

## 10. Add the Private Key to Project
Place your .p8 file inside:
```
secrets/Api_key.p8
```
Add secrets/Api_key.p8 to .gitignore (probably it will directly be added in `.gitignore`)

<hr data-start="378" data-end="381" class="">

## 7. Local .env for Fastlane
In fastlane/.env, create:
```
APPSTORE_KEY_ID=your-key-id
APPSTORE_ISSUER_ID=your-issuer-id
```

<hr data-start="378" data-end="381" class="">

## 11. Fastfile
```yaml
default_platform(:ios)

platform :ios do
  desc "Push a new beta build to TestFlight"
  lane :beta do |options|
    # Flexible parameter resolution: CLI option → ENV fallback
    key_id = options[:appstore_key_id] || ENV["APPSTORE_KEY_ID"]
    issuer_id = options[:appstore_issuer_id] || ENV["APPSTORE_ISSUER_ID"]
    key_filepath = options[:key_filepath] || ENV["KEY_FILEPATH"] || "./secrets/Api_key.p8"
    match_password = options[:match_password] || ENV["MATCH_PASSWORD"]
    match_git_basic_authorization = options[:match_git_basic_authorization] || ENV["MATCH_GIT_BASIC_AUTHORIZATION"]

    # Prepares the Fastlane environment for use in CI systems
    setup_ci(
      provider: "circleci"
    )

    # Create an App Store Connect API key using either passed options or environment variables
    api_key = app_store_connect_api_key(
      key_id: key_id,
      issuer_id: issuer_id,
      key_filepath: key_filepath
    )

    # Downloads code signing credentials using the API key
    match(
      type: "appstore",
      app_identifier: "yourAppBundleIdentifier",
      readonly: true,
      api_key: api_key,
      git_basic_authorization: match_git_basic_authorization
    )

    # Retrieves the most recent TestFlight build number
    latest = latest_testflight_build_number(
      app_identifier: "yourAppBundleIdentifier",
      api_key: api_key
    )

    # Increment build number locally
    increment_build_number(
        build_number: latest + 1,
        xcodeproj: "iosApp/iosApp.xcodeproj"
    )

    # Build ios app
    build_ios_app(
        scheme: "iosApp",
        project: "iosApp/iosApp.xcodeproj",
        output_name: "DeployIosApp.ipa",
        output_directory: "iosApp/build"
    )

    # Upload to TestFlight
    pilot(
      api_key: api_key,
      skip_waiting_for_build_processing: true
    )
  end
end
```

<hr data-start="378" data-end="381" class="">

## 12. Test Locally
```
bundle exec fastlane ios beta
```
<hr data-start="378" data-end="381" class="">

## deploy-ios.yml (Caller)
In your project .github/workflows/deploy-ios-caller.yml
```yaml
name: Deploy iOS to TestFlight via Composite Action

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: macos-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Use Deploy iOS Action
        uses: openMF/mifos-x-actionhub-publish-ios-on-appstore-production@main
        with:
          ruby-version: '3.3.5'
          appstore-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          appstore-issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          appstore-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
          match-password: ${{ secrets.MATCH_PASSWORD }}
          match-git-basic-auth: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
```

<hr data-start="378" data-end="381" class="">

## GitHub Secrets Required

Secret | Description
-- | --
APPSTORE_KEY_ID | From App Store Connect API
APPSTORE_ISSUER_ID | From App Store Connect API
APPSTORE_PRIVATE_KEY | Paste full .p8 content here
MATCH_PASSWORD | Password for encrypted match repo
MATCH_GIT_BASIC_AUTHORIZATION | e.g. username:token in base64 for match repo access


