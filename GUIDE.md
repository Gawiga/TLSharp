# Getting Start
- [Getting Start](#getting-start)
  - [Quick Configuration](#quick-configuration)
  - [First requests](#first-requests)
  - [Working with files](#working-with-files)
  - [Connecting with Telegram](#connecting-with-telegram)
  - [Get a user, create a group and send a invitation link](#get-a-user-create-a-group-and-send-a-invitation-link)

## Quick Configuration
Telegram API isn't that easy to start. You need to do some configuration first.

1. Create a [developer account](https://my.telegram.org/) in Telegram. 
1. Goto [API development tools](https://my.telegram.org/apps) and copy **API_ID** and **API_HASH** from your account. You'll need it later.

## First requests
To start work, create an instance of TelegramClient and establish connection

```csharp 
   var client = new TelegramClient(apiId, apiHash);
   await client.ConnectAsync();
```
Now you can work with Telegram API, but ->
> Only a small portion of the API methods are available to unauthorized users. ([full description](https://core.telegram.org/api/auth)) 

For authentication you need to run following code
```csharp
  var hash = await client.SendCodeRequestAsync("<user_number>");
  var code = "<code_from_telegram>"; // you can change code in debugger

  var user = await client.MakeAuthAsync("<user_number>", hash, code);
``` 

Full code you can see at [AuthUser test](https://github.com/sochix/TLSharp/blob/master/TLSharp.Tests/TLSharpTests.cs#L70)

When user is authenticated, TLSharp creates special file called _session.dat_. In this file TLSharp store all information needed for user session. So you need to authenticate user every time the _session.dat_ file is corrupted or removed.

You can call any method on authenticated user. For example, let's send message to a friend by his phone number:

```csharp
  //get available contacts
  var result = await client.GetContactsAsync();

  //find recipient in contacts
  var user = result.Users
	  .Where(x => x.GetType() == typeof (TLUser))
	  .Cast<TLUser>()
	  .FirstOrDefault(x => x.Phone == "<recipient_phone>");
	
  //send message
  await client.SendMessageAsync(new TLInputPeerUser() {UserId = user.Id}, "OUR_MESSAGE");
```

Full code you can see at [SendMessage test](https://github.com/sochix/TLSharp/blob/master/TLSharp.Tests/TLSharpTests.cs#L87)

To send message to channel you could use the following code:
```csharp
  //get user dialogs
  var dialogs = (TLDialogsSlice) await client.GetUserDialogsAsync();

  //find channel by title
  var chat = dialogs.Chats
    .Where(c => c.GetType() == typeof(TLChannel))
    .Cast<TLChannel>()
    .FirstOrDefault(c => c.Title == "<channel_title>");

  //send message
  await client.SendMessageAsync(new TLInputPeerChannel() { ChannelId = chat.Id, AccessHash = chat.AccessHash.Value }, "OUR_MESSAGE");
```
Full code you can see at [SendMessageToChannel test](https://github.com/sochix/TLSharp/blob/master/TLSharp.Tests/TLSharpTests.cs#L107)

## Working with files
Telegram separate files to two categories -> big file and small file. File is Big if its size more than 10 Mb. TLSharp tries to hide this complexity from you, thats why we provide one method to upload files **UploadFile**.

```csharp
	var fileResult = await client.UploadFile("cat.jpg", new StreamReader("data/cat.jpg"));
```

TLSharp provides two wrappers for sending photo and document

```csharp
	await client.SendUploadedPhoto(new TLInputPeerUser() { UserId = user.Id }, fileResult, "kitty");
	await client.SendUploadedDocument(
                new TLInputPeerUser() { UserId = user.Id },
                fileResult,
                "some zips", //caption
                "application/zip", //mime-type
                new TLVector<TLAbsDocumentAttribute>()); //document attributes, such as file name
```
Full code you can see at [SendPhotoToContactTest](https://github.com/sochix/TLSharp/blob/master/TLSharp.Tests/TLSharpTests.cs#L125) and [SendBigFileToContactTest](https://github.com/sochix/TLSharp/blob/master/TLSharp.Tests/TLSharpTests.cs#L143)

To download file you should call **GetFile** method
```csharp
	await client.GetFile(
                new TLInputDocumentFileLocation()
                {
                    AccessHash = document.AccessHash,
                    Id = document.Id,
                    Version = document.Version
                },
                document.Size); //size of fileChunk you want to retrieve
```

Full code you can see at [DownloadFileFromContactTest](https://github.com/sochix/TLSharp/blob/master/TLSharp.Tests/TLSharpTests.cs#L167)

## Connecting with Telegram

```csharp
var apiId = 0; //your apiId
var apiHash = "333ddd"; //your hash
var directoryInfo = new DirectoryInfo("D:\\"); //putting in a different place the .dat file
var store = new FileSessionStore(directoryInfo);
var session = store.Load("anonimous"); //if the already exists
string sessionName;

if (session == null)
{
    sessionName = "new"; //create new .dat file
}
else
{
    sessionName = session.SessionUserId.ToString();
}
var client = new TelegramClient(apiId, apiHash, store, sessionName);

await client.ConnectAsync(); //waiting for the connection
var isConnected = client.IsConnected;
var autorizado = client.IsUserAuthorized();

if (!autorizado)
{
    var telefone = "55511999990000"; // your telephone
    var hash = await client.SendCodeRequestAsync(telefone);
    var code = "0000"; //the code sended to you by telegram. you can change code in debugger
    var tlPassword = new TLPassword();
    var user = new TLUser();

    try
    {
        user = await client.MakeAuthAsync(telefone, hash, code);
    }
    catch (CloudPasswordNeededException)
    {
        //if you have 2fa auth you should have this
        var password = await client.GetPasswordSetting();
        var passwordStr = "password";
        user = await client.MakeAuthWithPasswordAsync(password, passwordStr);
    }
}
```


## Get a user, create a group and send a invitation link 

```csharp

var result = await Client.GetContactsAsync();

//gets a specific user to create group
var user = result.Users.Where(x => x.GetType() == typeof(TLUser))
    .Cast<TLUser>().FirstOrDefault(x => x.Phone == "551199009000");

//set the new user
var newUser = new TLInputUser() { UserId = user.Id };

//create a list of users
var listUser = new TLVector<TLAbsInputUser>();
var inputUser = new TLInputUser() { UserId = user.Id, AccessHash = (long)user.AccessHash };
listUser.Add(inputUser);

//make the resquest to create a new group
var newChatRequest = new TLRequestCreateChat()
{
    Users = listUser,
    Title = "Conan The Barbarian"
};
var newGroup = await Client.SendRequestAsync<TLUpdates>(newChatRequest);

//gets the group create
var absChat = newGroup.Chats.FirstOrDefault();
var newChat = (TLChat)absChat;

//gets the invite from that group
var inviteRequest = new TLRequestExportChatInvite() { ChatId = newChat.Id };
var chatInvite = await Client.SendRequestAsync<TLChatInviteExported>(inviteRequest);

//sends invite in a message
await Client.SendMessageAsync(new TLInputPeerUser() { UserId = user.Id }, chatInvite.Link.ToString());
