WearablePack-Myo
================

Salesforce and Thalmic Labs have come together to create a touchless surgery device. During surgery a Dr. will use a Myo controller and a Salesforce.com intergration to view images (X-Rays), order an additonal post op x-rays, notifly the post op team about the end of surgery, and send notifications to others.

![Flow Chart](https://cloud.githubusercontent.com/assets/6456976/3211270/82bf990c-ef0c-11e3-88f0-d9a2f4834aad.png)

##Architecture Components
There are three architectural componnets to the solution:

-  **The Connected Device - Thalmic Labs' Myo** -  https://www.thalmic.com/en/myo/

The Myo is a wearable device that is able to detect hand/spacial gestures and uses a bluetooth connection to allow the wearer to control an application.  In this usecase, the specific value for the surgeon is the touchless control, which preserves the clean environment needed for surgery.

- **The connected App - Unity 3D**  http://unity3d.com/

Unity 3D is a powerful platform for building applications iwth 3D interfaces. Traditionally, its origin is deeply rooted in game development. However, icreasingly, Unity3D is being used as the development platform of choice by 3D and spacially aware wearable devices such as the Myo. The Myo SDK also comes with a plug in for Unity for rapid 3D interface development.

- **The Salesforce1 Platform** - https://developer.salesforce.com/platform/overview

The Salesforce1 platform transforms the Myo Sergeon App in to a connected app, feeding it with data (patient info, Xrays, procedure details) as well as exposing business processes and workflows directly within the app.

##Integration Architectue 
There is two-way communication between Myo and the Unity3D connected app as well as another two-way comminucation beween Unity3D and the Salesforce1 platform.

![Flow Chart](https://cloud.githubusercontent.com/assets/2077602/3217042/32ad59c0-efdb-11e3-8b08-d005d24ae079.png)


###  Myo <=> Unity3D integration:
The Myo SDK includes a Unity Package that provides the connection between Unity3D and the Myo device.  Within this package are the components required to access the gyroscope and accelerometer as well as the five preset gestures.  This package is provided as part of the Myo Alpha 6 SDK.

###  Unity3D <=> Salesforce.com Integration:
This project illustrates three ways in which Salesforce.com and Unity3D can be integrated. Lets cover each one briefly:

   1. **Data Integration using the REST API and OAuth**
   The demo video shows X-Ray images as well as procedure details being rendered into the Unity 3D enviornment. This is achieved by retrieving the images and data from Salesforce using the REST API. The images are stored in Salesforce1 as attachments on the medical procedure (Case) 

   2. **User Interface Integration using Visualforce**
   The patient info panel on the left hand side of the user interface is a visualforce page being rendered directly within the 3D enviornments. This is possible by the Coherent UI add-on for Unity3d, which provides the ability to render web oages within the 3D environment. 

   3. **Data Integration via Apex Remote Methods and javascript**
   Coherent UI also provides a two way communication between the embedded browser and the Unity3D scene via Javascript. This two-way binding between the loaded web page (visualforce) and the Coherent UI/Unity3D environment was used to invoke Visualforce  methods that make use of remote actions. E.g. Creating the Post-Op reminder.

## Integration deep dive
The first example here is establishing a REST call (refer to the salesforce.cs file in the Unity3D folder uner src: src/Assets/scripts/salesforce.cs):

      public void query(string q){
            string url = instanceUrl + "/services/data/" + version 
									 + "/query?q=" + WWW.EscapeURL(q);

			WWWForm form = new WWWForm();			
			Hashtable headers = form.headers;
			headers["Authorization"] = "Bearer " + token;
			headers["Content-Type"] = "application/json";
			headers["Method"] = "GET";
			headers["X-PrettyPrint"] = 1;
			WWW www = new WWW(url, null, headers);

			request(www);
	 }

Here is an example of the client code that uses it (refer to the SalesforceClient.cs file in the same folder):

    //init object
    sf = gameObject.AddComponent<Salesforce>();
            
    // login
    sf.login(username, password + securityToken);
            
    // wait for Auth Token
    while(sf.token == null)
       yield return new WaitForSeconds(0.1f);
    
    // query: Retrieve the next closest scheduled surgery / procedure for current patient
    sf.query("SELECT Id, Surgery_Type__c FROM Case WHERE ContactId = '" +
                     patientID + "' ORDER BY Surgery_Date__c LIMIT 1");


Now, lets look at the Visualforce and Apex remote method integration approach. Firstly, this is the code that binds a method in Unity3D (C#) to a javascrip method (refer to SFDCcouiBinder.cs in the same folder):

	// calls a VF Page's javascript wrapper method, which in turn performs 
    // a saleesfroce remote action	
	public void orderPostOpXray(string procedureID){
       m_View.View.TriggerEvent ("orderPostOpXray", procedureID); 
	}

OrderPostOpXray is an vent that is captures in javascript within the Visualforce page. These events from within Unity3D are captured by including a Coherent UI javascriptt library in the Visualforce page:


	<apex:page controller="SFDCcouiBinderController" sidebar="false" 
     showHeader="false" standardStylesheets="false" docType="html-5.0">
    
        <apex:includeScript value="{!$Resource.jQuery2}"  />  
        <apex:includeScript value="{!$Resource.coherentUI}"  />
    
        <script type="text/javascript">
            j$ = jQuery.noConflict();
            j$(document).ready(function() {
                // Bind unity3d COUI events to a js functions
                 engine.on('orderPostOpXray',orderPostOpXray);
            });
        
            function orderPostOpXray(procedureID) {
                // invoke apex remote method
                Visualforce.remoting.Manager.invokeAction(
                    '{!$RemoteAction.SFDCcouiBinderController.orderPostOpXray}',
                    procedureID, 
                    function(result, event){
                    ...... Handle the response and call-back in to Unity3D !
                    
                    // callback a Unity3D exposed method
                    engine.call("callbackMethod",
                     			  String(result.Request_Post_Op_Xray__c)); 
                    },
                    {escape: true}
                );
            }
        
     	</script>
    
        <div id="responseErrors"></div>
	</apex:page>

Finally, to make a method in Unity3D available for the VIsualforce page to call (e.g.

###Required Unity3D Add-ons
**Playmaker** http://www.hutonggames.com/
This add-on is a visual scripter, providing a declarative development capability within Unity3D.  Playmaker provides the ability to quickly prototype and build your Unity scenes without the need to write extensive code.

