# TripPin Part 1 - Data Connector for an OData Service
This multi-part tutorial covers the creation of a new data source extension for Power Query. The tutorial is meant to be done sequentially � each lesson builds on the connector created in previous lessons, incrementally adding new capabilities to your connector. 

In this lesson, you will:
* Create a new Data Connector project using the Visual Studio SDK
* Author a base function to pull data from a source
* Test your connector in Visual Studio
* Register your connector in Power BI Desktop

## Creating a Basic OData Connector
In this section, we will create a new Data Connector project, provide some basic information, and test it in Visual Studio.

Open Visual Studio, and create a new Project. Under the Power Query folder, select the Data Connector project. For this sample, we will assume the project name is `TripPin`.

![VSProject](../../../blobs/vs2017_project.png)

Open the TripPin.pq file and paste in the following connector definition. It contains:
* A Data Source definition record for the TripPin connector
* A declaration that Implicit (Anonymous) is the only authentication type for this source
* A function (TripPinImpl) with an implementation that calls [OData.Feed](https://msdn.microsoft.com/library/mt260868.aspx)
* A shared function (TripPin.Feed) that sets the parameter type to Uri.Type
* A Data Source publishing record that will allow the connector to appear in the Power BI Get Data dialog 

```
section TripPin;

[DataSource.Kind="TripPin", Publish="TripPin.Publish"]
shared TripPin.Feed = Value.ReplaceType(TripPinImpl, type function (url as Uri.Type) as any);

TripPinImpl = (url as text) =>
    let
        source = OData.Feed(url)
    in
        source;

// Data Source Kind description
TripPin = [
    Authentication = [
        Implicit = []
    ],
    Label = "TripPin Part 1 - OData"
];

// Data Source UI publishing description
TripPin.Publish = [
    Beta = true,
    Category = "Other",
    ButtonText = { "TripPin OData", "TripPin OData" }
];


```

Open the TripPin.query.pq file. Replace the current contents with a call to your exported function and.

```
TripPin.Feed("http://services.odata.org/v4/TripPinService/")
```

Click the Start button to launch the M Query utility. 

> The `<project>.query.pq` file is used to test out your extension without having to deploy it to your Power BI Desktop's bin folder. Clicking the Start button (or pressing F5) will automatically compile your extension and launch the M Query utility.

Running your query for the first time will result in a credential error. 
In Power Query, the hosting application would convert this error into a credential prompt.
In Visual Studio, you will receive a similar prompt that calls out which data source is missing credentials and its data source path.
Select shortest of the data source paths (http://services.odata.org/) � this will apply your credential to all URLs under this path.
Select the Anonymous credential type, and click Set Credential. 

![Credential prompt](../../../blobs/credentialPrompt.png)

Click OK to close the dialog, and then press the Start button once again. You see a query execution status dialog, and finally a Query Result table showing the data returned from your query.

![Query results](../../../blobs/trippin1Results.png)

You can try out a few different OData URLs in the test file to see what how different results are returned. For example:
1. http://services.odata.org/v4/TripPinService/Me
2. http://services.odata.org/v4/TripPinService/GetPersonWithMostFriends()
3. http://services.odata.org/v4/TripPinService/People 

The `TripPin.query.pq` file can contain single statements, let statements, or full section documents. 

```
let
    Source = TripPin.Feed("http://services.odata.org/v4/TripPinService/"),
    People = Source{[Name="People"]}[Data],
    SelectColumns = Table.SelectColumns(People, {"UserName", "FirstName", "LastName"})
in
    SelectColumns
```

Open [Fiddler](http://www.telerik.com/fiddler) to capture HTTP traffic, and run the query. You should see a few different requires to services.odata.org, generated by the mashup container process. You can see that accessing the root URL of the service results in a 302 status and a redirect to the longer version of the URL. Following redirects is another behavior you get �for free� from the base library functions. 
One thing to note if you look at the URLs is that you can see the query folding that happened with the `SelectColumns` statement.
`http://services.odata.org/v4/TripPinService/People?$select=UserName%2CFirstName%2CLastName`

If you add more transformations to your query, you will see how they impact the generated URL. 

> This behavior is important to note. Even though you did not implement explicit folding logic, your connector inherits these capabilities from the [OData.Feed](https://msdn.microsoft.com/en-us/library/mt260868.aspx) function. M statements are compose-able � filter contexts will flow from one function to another, whenever possible. This is similar in concept to the way data source functions used within your connector inherit its authentication context and credentials. In later lessons, we will replace the use of [OData.Feed](https://msdn.microsoft.com/en-us/library/mt260868.aspx), which has native folding capabilities, with [Web.Contents](https://msdn.microsoft.com/library/mt260892.aspx), which does not. To get the same level of capabilities, we will need to use the Table.View interface and implement our own explicit folding logic.

## Loading Your Extension in Power BI Desktop
To use your extension in Power BI Desktop, you'll need to copy your connector project's output file (TripPin.mez) to your Custom Connectors directory. 
1. In Visual Studio, select **Build | Build Solution (F6)** from the menu bar. This will generate the .mez file for you project. By default, this will go in your project's bin\Debug folder. 
2. Create a `[My Documents]\Microsoft Power BI Desktop\Custom Connectors` directory
3. Copy the extension file into this directory
4. Enable the **Custom data connectors** preview feature in Power BI Desktop (under *File | Options and settings | Custom data connectors*)
5. Restart Power BI Desktop
6. Click **Get Data | More �** to bring up the Get Data dialog

You should be able to locate your extension by typing its name into the search box. 

![Get Data Dialog](../../../blobs/trippin1GetData.png)

Double click on the function name and the function invocation dialog will appear. Enter the root URL of the service (http://services.odata.org/v4/TripPinService/), and click OK. 

![Invoke Function](../../../blobs/trippin1Function.png)

Since this is the first time you are accessing this data source, you will receive a prompt for credentials. Check that the shortest URL is selected, and click Connect. 

![Credential prompt](../../../blobs/trippin1Creds.png)

Notice that instead of getting a simple table of data, the navigator appears. This is because the [OData.Feed](https://msdn.microsoft.com/en-us/library/mt260868.aspx) function returns a table with special metadata on top of it that the Power Query experience knows to display as a navigation table. We will cover how you can create and customize your own navigation table in a future lesson. 

![Nav Table](../../../blobs/trippin1NavTable.png)

Select the **Me** table, and click Edit. Notice that the columns already have types assigned (well, most of them). This is another feature of the underlying [OData.Feed](https://msdn.microsoft.com/library/mt260868.aspx) function. If you watch the requests in [Fiddler](http://www.telerik.com/fiddler), you will see that we fetch the service's $metadata document. The engine's OData implementation does this automatically to determine the service's schema, data types, and relationships. 

![Me Record](../../../blobs/trippin1Me.png)

## Conclusion

This lesson walked you through the creation of a simple connector based on the [OData.Feed](https://msdn.microsoft.com/library/mt260868.aspx) library function. As you saw, very little logic is needed to enable a fully functional connector over the OData base function. Other extensibility enabled functions, such as [ODBC.DataSource](https://msdn.microsoft.com/library/mt708843.aspx), provide similar capabilities. 
In the next lesson, we will replace the use of [OData.Feed](https://msdn.microsoft.com/library/mt260868.aspx) with a less capable function - [Web.Contents](https://msdn.microsoft.com/library/mt260892.aspx). Each lesson will implement more connector features, including paging, metadata/schema detection and query folding to the OData query syntax, until your custom connector supports the same range of capabilities as [OData.Feed](https://msdn.microsoft.com/en-us/library/mt260868.aspx).