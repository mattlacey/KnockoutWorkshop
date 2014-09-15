#A Fast Account Rolodex With KnockoutJS

##Step 1: Create a Visualforce Page That Uses the Account Standard Controller

Creating a basic Visualforce page that will be Salesforce1 friendly is very easy, first we need to create a new page and turn off the header, sidebar and standard styling.

1. Launch your browser and go to https://login.salesforce.com.

2. Enter your username (in the form of an email address) and password.

3. Click on your name, and then choose *Developer Console* from the menu.

4. In the Developer Console, choose *File -> New -> Visualforce Page* and then enter a name for the page such as 'Rolodex'.

5. Replace the initial code with the following:


    ```Visualforce
    <apex:page standardController="Account" showHeader="false" sidebar="false" standardStylesheets="false">

    	<h1><apex:outputText value="{!Account}"/></h1>

    </apex:page>
    ```

6. Make the page easy to access by creating a new custom button on the Account object, via  *Setup -> Customize -> Accounts -> Buttons, Links, and Actions* and then *New Button or Link*.
![Account Settings](./Images/AccountSettings.png "Account Buttons, Links and Actions")

7. Fill in the button details as in the image below, and click *Save*. Your Visualforce page will be available because we used the Account standard controller.
![Custom Button Settings](./Images/CustomButton.png "Use these settings for your button")

8. Finally, navigate to the Accounts tab and choose an Account. Then click *Edit Layout* to open the page layout editor

![Editing The Page Layout](./Images/EditLayout.png "This is the link to edit a page layout")

and then drag the new custom button into the custom buttons area and click *Save*.

![Editing The Page Layout - Adding The Button](./Images/AddingCustomButton.png "Drag the button to the custom buttons area")

If you click the button you'll now see your exciting new page... that shows the Account ID. Time to make it a bit more interesting!

##Step 2: Getting Started With KnockoutJS

KnockoutJS uses a Model-View-View Model pattern, which means everything is split neatly into three areas. The view is HTML/Visualforce markup with some extra parameters, it displays data and sends commands to the View Model. The View Model is all code, and represents the data for the View and provides the operations that can be performed on it. Data that is stored by the application comes under the Model component, and in this scenario that consists of our SObjects in the database.

The three basic things we need to do to start with are to create a View Model in Javascript, create a View using HTML, and then fire up KnockoutJS to hook the two together.

1. Before doing anything else we need to include the KnockoutJS source in our page, and to make that easy we'll use a CDN (Content Delivery Network) hosted version, so add the following line to the Rolodex page above right after the `<apex:page>` opening tag:

    <script src="//cdnjs.cloudflare.com/ajax/libs/knockout/3.1.0/knockout-min.js"/>

2. Now we'll add a super simple View Model, so under the last line we'll add our own Javascript which will be the start of our Rolodex View Model. For now we'll just add two members to our model, and populate them with values from the account record using standard Visualforce notation:

	```javascript
	<script type="text/javascript">

		function rolodexModel()
		{
			this.accountId = '{!Account.Id}';
    		this.accountName = '{!Account.Name}';
		}

	</script>
	```

3. The next step is to sort out the View part of the equation, that is the part that displays the data provided by the View Model. The view in this case is all of the markup in the page, so replace the old line with the `<h1>` tags with the following:

	```
	<div class="panel">
		<h1 data-bind="text: accountName"></h1>
	</div>
	```

 As you can probably guess at this point KnockoutJS uses a special attribute called `data-bind` to bind parts of the View to the data in the View Model. In this case, we're using the `text` binding variant to use the `accountName` property of the View Model as the content for the `<h1>` tags.

4. If you've viewed the page at this point, you won't have seen anything on the screen, and that's because we haven't connected to the View Model to it's view. We do this with simple call to the `applyBindings` method provided by KnockoutJS. This method takes one parameter which is an instance of the View Model. We want this link to be created when the page has loaded, so we'll create a quick one-line function and assign it to `window.onload` right before the closing script tag.

	```
    window.onload = function() { ko.applyBindings(new rolodexModel()) };
    ```

5. Navigate to the page from an account using the custom button created in step 1 and you should see the account's name on the screen. If not, check your browser's developer tools console for any javascript errors. The page is pretty plain at this point, so add the following line underneath the opening page tag to include some pre-prepared CSS.

	```
	<link rel="//dl.dropboxusercontent.com/u/18791682/DF/dfko.css"/>
	```


##Step 3: Building On The Basics

Now we've got a the framework in place we'll start expanding our setup, and modify the page so that it lists all of the contacts associated with an account (with a sensible limit of 100 for example purposes). We could just do this with Visualforce binding, but to keep things nice and clean we'll load the contacts using Javascript remoting, with a very simple Apex class providing the list of contacts with some chosen fields.

1. Before making further changes to the page we'll first create the Apex class that will provide data from our model. In the Salesforce Developer Console choose *File -> New -> Apex Class* and then specicy a name, e.g. 'RolodexRemoting'. Replace the default class content with this code, which is a relatively standard starting point for an extension controller, with the small exception that the class has `global` scope, which is required for remoting classes. The constructor doesn't have anything in it's body as we're not going to use it, we merely need this constructor so that Visualforce will let us use the class as an extension controller alongside the Account standard controller the page uses at present. 

	```
	global class RolodexRemoting
	{
	    global RolodexRemoting(ApexPages.StandardController sc) {}
	}
	```

2. Next, we need a remoting method that returns the contacts and fields we're interested in. For this exercise this is a one-liner, and simply returns the result of a SOQL query as a list of Contact records. It is static (as all remoting methods must be), and pulls the Id of the account being used from the page parameters.

	```
    @RemoteAction
    global static List<Contact> LoadContacts()
    {
        Id accountId = ApexPages.currentPage().getParameters().get('id');
        return [select Id, Name, Email, Phone
                from Contact where AccountId = :accountId
               	order by LastName asc];
	}
	```

3. Once the class has been saved, switch back to the page source and add our new Apex class as an extension controller using the `extensions` attribute in the opening page tag:

	```
	<apex:page standardController="Account" extensions="RolodexRemoting" showHeader="false" sidebar="false" standardStylesheets="false">
	```

4. Now we need to modify the View Model so that it pulls a list of contacts from the controller, but to do that we'll need somewhere to store them in the model. Obviously an array is required, but because this will be something that changes over time (using remoting means we'll load the contacts after the page has loaded) we need to use what's known as an *observable array*. Observables and observable arrays are mechanisms provided KnockoutJS that creates a dynamic, two-way binding between the View and View Model. This means if a value changes in the View Model it will be repfected in the view, and vice versa. Modify the View Model so that it looks like this:

	```
	function rolodexModel()
	{
		var self = this;

		self.accountId = '{!Account.Id}';
		self.accountName =  '{!Account.Name}';

		self.contacts = ko.observableArray();
	}
	```

	*Note:* a property has been created called `self` so that in any code added to the model we have an easy way to reference the model itself. Using `this` can get particularly tricky in Javascript when dealing with callbacks and other idioms.

5. Assigning values to observables is easy, you simply pass it as a parameter, so if we had an observable called age, defined using `self.age = ko.observable;`, it could be set to a value such as 40 by doing `self.age(40);`. The same goes for observable arrays, so to set an array called people we could do `self.people(['Alice', 'Bob', 'Carol']);`.
The upshot of this is that because we our Apex method returns a list of contacts, we can simply assign the result straight to our contacts observable array. Before the closing brace of the view model, add the code below. This calls the method provided by the extension controller, which uses a callback function that runs when the request completes. The `result` parameter will be an array of records, and we use this data without modification.

	```
	RolodexRemoting.LoadContacts(self.accountId, function(result, event)
	{
		self.contacts(result);
	});
	```

6. Now we'll display these contacts under the account name by updating our view. To loop over the contacts we need to use another binding variant, this time we use `foreach`. This can be applied to any element (in this case a div) and then the contents of that element, including bindings, are repeated and evaluated for each item in the array. Add the following code after the `<h1>` tags and reload the page, you should see the account name as before, and then after a short delay the list of contacts will appear when the remoting call completes.

	```
	<div data-bind="foreach: contacts">
		<div class="record">
			<h2 data-bind="text: Name"/>
		</div>
	</div> 
	```

In this code, the "Name" bound to the text for the h2 element is the name of the field to display from the contact.













