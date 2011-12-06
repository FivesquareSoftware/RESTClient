### Usage


#### Creating Resources


_You can set up a base resource for other resources to inherit, or to use directly_

```objective-c 
RCResource *site = [RCResource withURL:@"http://example.com"];
// Health check
[site getWithCompletionBlock:^(RCResponse *response){
	if (NO == response.success) {
		NSLog(@"The site is down!");
	}
}];
```

_You can configure settings on parent resources (they will be inherited by child resources)_

```objective-c 
site.timeout = 20;
site.headers = [NSDictionary dictionaryWithObjectsAndKeys:
				@"foo",@"X-MyCustomHeader"
				, nil];

site.progressBlock = ^(NSDictionary *progressInfo){
	mySpinner.animating = [[progressInfo objectForKey:kRESTClientProgressInfoKeyProgress] floatValue] < 1.f;
};
```

_Create child resources using the -resource: method with a relative path (which can be any object that will output a useful description)_

```objective-c 
RCResource *users = [site resource:@"users"];
RCResource *posts = [site resource:@"posts"];
RCResource *one = [site resource:[NSNumber numberWithInt:1]];
```

_resourceWithFormat: allows you to use format strings to create child resources_

```objective-c 
RCResource *user123 = [site _resourceWithFormat:@"users/%@",[NSNumber numberWithInt:123]];
RCResource *todaysPosts = [site _resourceWithFormat:@"posts/%@",@"today"];
```

_Children can have children, ad infinitum..._

```objective-c 
 RCResource *uploads = [posts resource:@"uploads"];
 RCResource *myUploads = [uploads resource:@"123"];
```

_Child resources can override settings inherited from parent resources_

```objective-c 
// Give user more time to upload
uploads.timeout = 60;
```

_Create single resources anonymously if you just want to use them once_

```objective-c 
RCResponse *response = [[site resource:@"accounts/abc123"] get];
```

	

#### Making Requests



_Call HTTP methods on resources with blocks to run the requests asynchronously_

```objective-c 
[posts getWithCompletionBlock:^(RCResponse *response){
	NSArray *result = response.result;
	for (id posts in result) {
		NSLog(post);
	}
}];
```

_Or, make synchronous requests to handle responses inline_


```objective-c 
RCResponse *response = [posts get];
NSLog(@"posts: %@",response.result);
```

_Blocks are well typed, so you can reuse them_

```objective-c 
RCCompletionBlock mappingBlock = ^(RCResponse *response) {
	MyMapper *mapper = [MyMapper withClass:[MyModelObject class]];
	MyModelObject *object = [mapper map:response.result];
	[object save];
};

[user123 getWithCompletionBlock:mappingBlock];
```

_Set up a preflight block to run on all requests for a resource. These are run on the main queue so UI code is OK. Returning NO will abort the request._

```objective-c 
RCResource *myProtectedResource = [site resource:@"protected"];
myProtectedResource.preflightBlock = ^(RCRequest * request){
	if (NO == [accountController authorized:request]) {
		NSDictionary *userInfo = [NSDictionary dictionaryWithObject:@"You need to log in to see this" 
															 forKey:NSLocalizedDescriptionKey];
		[errorReporter report:[NSError errorWithDomain:@"MyDomain" code:-1 userInfo:userInfo]];
		return NO;
	}
	return YES;
};
```

_Progress blocks allow you to do work when data is received on a connection. They run once when the request begins and are called on the main queue like preflight blocks._

```objective-c 
RCProgressBlock pictureProgressBlock = ^(NSDictionary *progressInfo){
	pictureDownloadProgressBar.hidden = NO;
	pictureDownloadProgressBar.progress = [[progressInfo objectForKey:kRESTClientProgressInfoKeyProgress] floatValue];
}	
[[pictures resource:@"abc123"] getWithProgressBlock:pictureProgressBlock completionBlock:^(RCResponse *response) {
	pictureDownloadProgressBar.hidden = YES;
}];
```

_Completion blocks run when a request is complete, regardless of whether it was a success or failure. Again, called on the main queue so you can interface safely with UI code._

```objective-c 
[posts getWithCompletionBlock:^(RCResponse *response) {
	if (NO == [response isOK]) {
		// it's safe to call UI code here
		UIAlertView *alert = [[UIAlertView alloc] initWith ...];
		[alert show];
	}
}];
```

_But you can also do background work from any of these blocks if you want, just by dispatching to another queue_

```objective-c 
[[posts resource:[NSNumber numberWithInt:1]] getWithCompletionBlock:^(RCResponse *response){
	if ([response isOK]) {
		// save something to your database in the background using a dedicated DB mapping queue
		
		dispatch_queue_t q_db_mapping = dispatch_queue_create("com.example.db_mapping", NULL);

		dispatch_async(q_db_mapping, ^(){
			id myObject = [MyObject new];
			[myObject mapValuesFromObject:response.result];
			[myObject save];

			dispatch_release(q_db_mapping);
		});
	}
}];
```

_You can transform response results before they are passed to completion blocks using post processing blocks. These are called on a concurrent queue to achieve the best possible throughput. Use these instead of completion blocks when you know you have significant work to do to process a response's data._

```objective-c 
RCResource *pictures = [site resource:@"pictures"];
pictures.postProcessorBlock = ^(id result) {
	// we know the service returns raw data, and we transform it to a custom type
	MyImageType *image = [MyImageType receiptWithData:(NSData *)result];
	return image; 
	// response.result is set to the custom image class instead of the orginal raw data when the block 
	returns and any completion block will only see the transformed result
};
[[pictures resource:@"abc123"] getWithCompletionBlock:^(RCResponse *response) {
	NSLog(@"Your picture is ready: %@",response.result);
}];
```


_Post processing blocks are an ideal match to Core Data's new blocks-based interfaces, allowing you to save data on a child context in the background and not worry about how it propagates to the main thread's context._

```objective-c 
childContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
childContext.parentContext = mainContext; // your main queue MOC, maybe in your appDelegate

// ...

pictures.postProcessorBlock = ^(id result) {
	MyModelObject *myObject = [NSEntityDescription insertNewObjectForEntityForName:someEntityName 
															inManagedObjectContext:childContext];	
	// process result into myObject ...
	NSError *error = nil;
    if ([childContext save:&error]) {
        __autoreleasing NSError *parentError = nil;    
        __weak NSManagedObjectContext *parent = childContext.parentContext;
        [parent performBlock:^{
            [parent save:&parentError];
        }];
    }
	// check errors ...
	return myObject;
};
```

_PUT and POST take payloads_

```objective-c 
NSDictionary *payload = [NSDictionary dictionaryWithObjectsAndKeys:
							@"me",@"username",@"test",@"password",nil];

[myAccount post:payload completionBlock:^(RCResponse *response) {
   if (NO == [response isOK]) {
	   // cound not create account 
   }
}];

RCResponse *response = [myAccount put:accountInformation];
if ([response isOK]) {
	NSLog(@"Updated myAccount: %@",response.result);
}
```


_You can download large files directly to disk_

```objective-c 
RCResource *images = [site resource:@"images"];
[images downloadWithProgressBlock:nil completionBlock:^(RCResponse *response) {
	NSURL *downloadedFile = [response.result valueForKey:kRESTClientProgressInfoKeyTempFileURL];
	// Move the file to where you want it
}
```

_Uploads can stream files directly from disk_

```objective-c 
RCResource *hero = [self.service resource:@"test/upload/hero"];

RCProgressBlock progressBlock = ^(NSDictionary *progressInfo){
	NSNumber *progress = [progressInfo valueForKey:kRESTClientProgressInfoKeyProgress];
	NSLog(@"progress: %@",progress);
};

RCCompletionBlock completionBlock = ^(RCResponse *response) {
	localResponse = response;
	dispatch_semaphore_signal(request_sema);
};

NSString *filePath = [[NSBundle mainBundle] pathForResource:@"t_hero" ofType:@"png"];
NSURL *fileURL = [NSURL fileURLWithPath:filePath];
[hero uploadFile:fileURL withProgressBlock:progressBlock completionBlock:completionBlock];
```



*RESTClient*


_Maybe you have non-RESTful services, and just need to handle the odd endpoint. You can use the RESTClient class directly._

```objective-c 
[RESTClient get:@"http://example.com/products.php?action=list" completionBlock:^(RCResponse *response){
	// do something with the response
}];

RCResponse *response = [RESTClient post:@"http://example.com/products.php?action=create" payload:productInformation];
```


*Request*


_If you need more control, you can create raw RCRequest objects directly_


```objective-c 
NSString *endpoint = [NSString stringWithFormat:@"%@/index",@"http://www.example.com"];
NSMutableURLRequest *URLRequest = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:endpoint]];
[URLRequest setHTTPMethod:kRESTClientHTTPMethodGET];

RCRequest *request = [[RCRequest alloc] initWithURLRequest:URLRequest];
__block RCResponse *localResponse = nil;

dispatch_semaphore_t request_sema = dispatch_semaphore_create(1);
dispatch_semaphore_wait(request_sema, DISPATCH_TIME_FOREVER);
[request startWithCompletionBlock:^(RCResponse *response) {
	localResponse = response;
	dispatch_semaphore_signal(request_sema);
}];
dispatch_semaphore_wait(request_sema, DISPATCH_TIME_FOREVER);
dispatch_semaphore_signal(request_sema);
dispatch_release(request_sema);
```

_Take a look at the class for more information_

