PhoneGap-PushSharp
==================

Get Stated with the Push Notification for Android and iOS on a .Net server and PhonegapBuild

iOS
First... Lets care about the certificates!

After the App is created and the push notification is enabled, you need to create a Certificate and a Provisional Profile that PhonegapBuild accepts as appropriate for push notifications.

CERTIFICATES

On the "Certificates" section you must create a -App Store and Ad Hoc- under the Distribution section.

  
[1.png]


In a MAC, in the Keychain Access you need to go to Certificate Assitant > Request a Certificate From a Certificate Authority
Make sure you checked the option saved in disk and that you specify key pair information (Key Size= 2048 bits, Algorithm=RSA).

This generates a file that by default is named as "CertificateSigningRequest.certSigningRequest". This file is the one you are going to Upload to the Developer center and
Generate your Certificate:



This process creates a .cer file, doble click that file so you can install it in your Keychain Access.
Once there, locate the new certificate and export it as a .p12 file (right click on the file > Export). 
Here you will need to provide a password (YOU WILL BE ASKED ABOUT IT LATER).

______________________________________________________________

PROVISIONING PROFILE:
On "Provisioning profiles" section, you need to create a new Provisioning profile:



Under "Distribution section", you need to select -Ad Hoc- and the App ID that matches the one you are about to deploy.



Last thing you need to do is to Select your certificate (the same certificate created before):



Now, you finally have a .p12 and a .mobileprovision, both needed in phonegapBuild!

With this configuration, you can now implement the push plugin (Here you can find an example! http://devgirl.org/2013/01/24/push-notifications-plugin-support-added-to-phonegap-build/), and your devices should be able to register on APNs servers.

______________________________________________________________

SERVER SIDE CONFIGURATION

Here comes the tricky part:

In this document we will go through the implementation using PushSharp (this library can be found in NuGet).
First you need to create a new Certificate following all the steps (only this time, you are not selecting App Store and Ad Hoc- under the Distribution section, but -Apple Push Notification service SSL (SandBox)- under development section.

*NOTE: *This new certificate is the one that will authenticate against APNs Servers. I created it with the same password, that way, I dont get confused about one - required by phonegapBuild, and the other one -require for SSL authentication.

After importing the library into your proyect, the code is as simple as this:

    

		try
            	{
                //Create our push services broker
                var push = new PushBroker();

                //Registering the Apple Service and sending an iOS Notification
                var appleCert = File.ReadAllBytes("this is the path of your Certificate.p12 file");
                push.RegisterAppleService(new ApplePushChannelSettings(appleCert, "This is the password of your certificat));
                push.QueueNotification(new AppleNotification()
                                           .ForDeviceToken(
                                               "This is the device token")
                                           .WithAlert("this is the message")
                                           .WithBadge(1)
                                           .WithSound("default"));
                push.StopAllServices(true); // I didn't get the popup message until I added this line, it seems to be closing the conection and flushing the payload.
            }
            catch (Exception ex)
            {
  		//WHATEVER YOU WANT TO DO WITH THE ERROR MESSAGE;
	    }

Android
The Client Side is the same, (no certificates required) you can also see it in the same example: http://devgirl.org/2013/01/24/push-notifications-plugin-support-added-to-phonegap-build/ 
You don't need to do anything extra to register you device on the GCM servers, in the example and the following steps are all the information you need to successfully register the device.

The Server Side:

First, you need to configure GCM (google cloud messaging) service:

Place yourself in https://code.google.com/apis/console/* *and sign in with the enterprise (thinking about a commercial app) gmail account and select Services:



Look for the Google Cloud Messaging for Android service, and enable it.



After that, you can see in the API Access section, your API Key:

In the Overview link, you can find the project number:



Finally, to connect with GCM servers, you can use the same library PushSharp, the following example was taken from their documentation:

		
		push.RegisterGcmService(new GcmPushChannelSettings("theauthorizationtokenhere"));
		//Fluent construction of an Android GCM Notification
		//IMPORTANT: For Android you MUST use your own RegistrationId here that gets generated within your Android app itself!
		push.QueueNotification(new GcmNotification().ForDeviceRegistrationId("DEVICE REGISTRATION ID HERE")
		.WithJson("{\"alert\":\"Hello World!\",\"badge\":7,\"sound\":\"sound.caf\"}"));
		
		
or you can do it manually, as I did:

	public void SendGcmPushNotification(PushGcmNotificationMessageDto pushDto)
        {
            string content = pushDto.MessageContent;
            string title = pushDto.MessageTitle;
            string postData = "{ \"registration_ids\": [";

            foreach (var registrationId in pushDto.RegistrationIdList)
            {
                postData += "\"" + registrationId + "\",";
            }

            var newLength = postData.Length - 1;
            postData = postData.Substring(0, newLength);
            postData += "], \"data\": {\"title\":\"" + title +
                                  "\", \"message\": \"" + content + "\"}}";

            SendGcmNotification(postData);
        }


	private void SendGcmNotification(string postData)
        {
            string apiKey = Common.Settings.SnapCauseSettings.Default.GCMApiKey;
            ServicePointManager.ServerCertificateValidationCallback += ValidateServerCertificate;

            byte[] byteArray = Encoding.UTF8.GetBytes(postData);
            HttpWebRequest.Headers.Add(string.Format("Authorization: key={0}", apiKey));
            HttpWebRequest.ContentLength = byteArray.Length;
            Stream dataStream = HttpWebRequest.GetRequestStream();
            dataStream.Write(byteArray, 0, byteArray.Length);
            dataStream.Close();


            try
            {
                var errorText = string.Empty;
                var response = HttpWebRequest.GetResponse();
                var responseCode = ((HttpWebResponse)response).StatusCode;
                if (responseCode.Equals(HttpStatusCode.Unauthorized) || responseCode.Equals(HttpStatusCode.Forbidden))
                {
                    errorText = "Unauthorized - need new token";
                }
                else if (!responseCode.Equals(HttpStatusCode.OK))
                {
                    errorText = "Response from web service isn't OK";
                }

                if (!string.IsNullOrEmpty(errorText))
                {
                    // TODO: WHAT SHOULD WE DO WHEN AN ERROR IS THROWN?
                }

                var reader = new StreamReader(response.GetResponseStream());
                reader.Close();
            }
            catch (Exception e)
            {
                // TODO: WHAT SHOULD WE DO WHEN AN EXCEPTION IS THROWN?
            }
        }
