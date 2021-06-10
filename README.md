# insurance-android-documentation
Documentation for the Android SDK for insurance

### The following will walk you through the installation steps

1. Get your github username added to our Jitpack account. *This is required to access the build artifacts (aar / jar files).*

2. Sign up for your own Jitpack account. (https://jitpack.io/).

3. Generate an authentication token for yourself on Jitpack.

4. Add the token to $HOME/.gradle/gradle.properties: `authToken=AUTHENTICATION_TOKEN`

5. Then use authToken as the username in your build.gradle: 
```gradle
repositories {
        maven {
            url "https://jitpack.io"
            credentials { username authToken }
        }
 }
 ```
6. Install the SDK in your android app as you would regularly:
```gradle
dependencies {
    implementation 'com.github.MoolahTech:insurance-android:$CURRENT_VERSION'
}
```
7. Build your app. If everything passes, you are ready to use the SDK!

To use the SDKs, you have to first get access to your access key and secret key. Request your SPOC for these credentials. Staging credentials will be given first, and then production credentials will be issued once a round of testing has been performed. The API urls will also be shared via your SPOC.
**The secret key must be kept secret**. Please make sure this key is not on the phone, or anywhere in your database or permanent storage. It must be kept in your live environment as a env. variable or use other similar key storage mechanisms like AWS KMS. Leakage of your secret key could compromise all your users and could lead to very bad things.

**IMPORTANT: The steps below also detail how to use the SDK to build your own UI. You can only build your own UI if you are a licensed distributor with a valid ARN number.** If you are not a distributor, you must use the standard SDK UI.
In the below documentation, we will use the `<YourUi>` symbol to denote how to build your own UI for the respective step.

### Flow

To use the SDK, the following flow must be followed:
1. Create user and get a user identity token to use in subsequent requests
2. Initialize the SDK with your API key and user identity token
3. Call the CreateInsurancePolicy Activity with appropriate parameters and subscribe to activity completion events
4. Subscribe to backend callbacks to receive updates on policy issuance
5. Call relevant APIs to get the insurance policy pdf and other details

## User creation

Every request from the SDK to the our API must be authenticated using your access key (which identifies the partner making the request) and an IDENTITY_TOKEN (which identifies the user making the request). A user must be created via a server-to-server call using your access and secret key. In return, an expiring token is passed back. Please store this token SAFELY, preferably in the android keystore or other similar storage mechanism and pass it to the SDK.

**User create call**

**POST** ($BASE_URL)**/partners/users**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be user: `user: { phone_number.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| phone_number | string | 9876543210 OR +919876543210 |
| email | string | test@example.com |
| first_name | string | Foo |
| last_name | string | Bar |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-USER-IDENTITY-TOKEN | string | abcd.efg.hijk |
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-hijk-xyz |

The identity token expires every 24 hours, so please make sure to have a refresh mechanism built in.

**Token refresh call**

**POST** ($BASE_URL)**/partners/users/new_token**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-PARTNER-SECRET-KEY | string | xyzab |

Params (root key must be user: `user: { uuid.... }`):
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-rf-rrrr |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-USER-IDENTITY-TOKEN | string | abcd.efg.hijk |
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| uuid | string | abcd-efg-hijk-xyz |

### Initialization

Initialize the SDK by calling `init`. Make sure to do this before calling any of the activities, or before using the SKD in any way:
```kotlin
InsuranceSDK.init(Context, PARTNER_ACCESS_KEY, USER_IDENTITY_TOKEN, isProduction: Boolean)
```

### Create policy purchase

Call `CreateInsurancePolicy` activity with following params:

| name | type | example | mandatory |
| ---- | ---- | ------- | :-------: |
| partnerTransactionId | string, generated by you, identifies the transaction | XYZ123 | true |
| productType | string | "job_loss", "covid" (complete list provided separately) | true |
| provider | string | "chola", "icici_lomabrd" (complete list provided separately) | true |
| coverageAmount | decimal | 50000.0 | false, but if passed in, make sure it is a valid value according to the table shared with you. Please note the user may change the coverage amount before transacting. This will be shared with you in the callbacks |

```kotlin
import `in`.savvyapp.insurance_android.CreateInsurancePolicy

val intent = Intent(activity, CreateInsurancePolicy::class.java)
intent.putExtra("partnerTransactionId", "9898989898")
intent.putExtra("productType", "job_loss")
intent.putExtra("provider", "chola")
this.startActivity(intent)
```
**Policy purchase success**: The result code is Activity.RESULT_OK. Expect callbacks on your backend.

**Policy purchase failure**: The result code is Activity.RESULT_CANCELED. There was an error during purchase, and user may not proceed. For these kinds of errors, we recommend logging them and reporting them along with the intent parameters. The intent will contain:
* `errorCode`
* `message`

Sample flow of the activity:

<p float="left">
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%201.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%203.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%208.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%209.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%202.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%2010.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%205.png" width="200" height="356" />
        <img src="https://github.com/MoolahTech/insurance-android-documentation/blob/main/Android%20-%204.png" width="200" height="356" />
</p>

## Callbacks

The insurance transaction can be in various states, which are communicated with you via callbacks.
1. New insurance transaction begun
2. Insurance transaction completed
3. Insurance transaction expired / abandoned

Each callback is sent with a hash string. This hash string is a pipe-joined string of all the params sent, which is then hashed using HMAC with SHA256 using your secret key. You **must** verify this hash at your end, otherwise attackers might simply be able to spoof requests to your open endpoints.

**New insurance transaction begun**
Params sent:
```ruby
      {
        transaction_type: 'insurance_purchase',
        transaction_id: <YOUR_ID>,
        status_is: 'pending',
        status_was: nil,
        coverage_amount: 1000,
        premium: 100,
        end_date: "31/12/2099",
        created_at: "01/01/2021",
        hash: <HASH STRING>
      }
```

The hash string is generated as follows:
```ruby
hash_string = "transaction_type|transaction_id|status_is|status_was|coverage_amount|premium|end_date|created_at"
hash = HMAC('sha256', hash_string, secret_key)
```

**Insurance transaction completed**
Params sent:
```ruby
      {
        transaction_type: 'insurance_purchase',
        transaction_id: <YOUR_ID>,
        status_is: 'completed',
        status_was: <PREVIOUS STATUS>,
        coverage_amount: 1000,
        premium: 100,
        end_date: "31/12/2099",
        created_at: "01/01/2021",
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'completed', 'error'`

**Insurance transaction error**
Params sent:
```ruby
      {
        transaction_type: 'insurance_purchase',
        transaction_id: <YOUR_ID>,
        status_is: 'error',
        status_was: <PREVIOUS STATUS>,
        coverage_amount: 1000,
        premium: 100,
        end_date: "31/12/2099",
        created_at: "01/01/2021",
        hash: <HASH STRING>
      }
```
The possible statuses are: `'pending', 'completed', 'error'`

### Get policy details and pdf

**GET** ($BASE_URL)**/partners/<provider(chola, icici_lombard etc)>/policies/<partnerTransactionId(ID passed in during creation)>**

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| X-PARTNER-ACCESS-KEY | string | abcde |
| X-USER-IDENTITY-TOKEN | string | xyzab |

_Response:_

Headers:
| name | type | example |
| ---- | ---- |:-------:|
| Content-Type | string | application/json |

Body:
| name | type | example |
| ---- | ---- |:-------:|
| payment_status | string | pending |
| coverage_amount | double | 50000.0 |
| end_date | date | 01/01/2025 |
| pdf_link | string | https://www.example.com |
