---
path: getting-sexy-with-core-data
date: 2015-10-29T14:27:49.262Z
title: "Getting Sexy with Core Data"
description: "Recently I was tasked with updating the way we we’re using Core Data in our application. The app was using the legacy concurrency type of NSConfinementConcurrencyType. When you use this to…"
---

Recently I was tasked with updating the way we we’re using _Core Data_ in our application. The app was using the legacy concurrency type of `NSConfinementConcurrencyType`. When you use this to initialize your NSManagedObjectContext you are _guaranteeing_ that you will access this context from the correct thread, if you do not, sorry Charlie it’s a **crash**. Some people still use this method and in fact if you call `[[NSManagedObjectContext alloc] init]`, this is the default concurrency type selected. You may have a single threaded application or do very minimal work on a background thread, and if so, then this could work for you. Sure, in the old setup it is possible to do a lot of work in the background, and in fact we do a lot of it. Currently we have a method that creates a new context for use on a background thread like:

```objective-c
- (NSManagedObjectContext *)backgroundContext {
  NSPersistentStoreCoordinator *coordinator = [self persistentStoreCoordinator];
  NSManagedObjectContext *bgContext = nil;
  if (coordinator != nil) {
    bgContext = [[NSManagedObjectContext alloc] init];
    [bgContext setPersistentStoreCoordinator:coordinator];
  }

  [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(objectContextDidSave:)  name:NSManagedObjectContextDidSaveNotification bgContext];

  return bgContext;
}
```

Then in objectContextDidSave: you merge in the changes to your main managed object context on the main thread. The problem with this, is that you can still perform disk IO on the main thread, blocking the UI, and potentially taking long enough that it is painfully noticeable.

Apple introduced a nice feature in iOS 5, _parent/child contexts_ as well as a concurrency model that _guarantees_ if you execute your code inside a block passed to performBlock: or performBlockAndWait: that it will use the correct queue. The new concurrency options you can initialize a NSManagedObjectContext with are NSPrivateQueueConcurrecyType and NSMainQueueConcurrencyType. When you select the NSPrivateQueueConcurrencyType and call performBlock: or performBlockAndWait: on this context it will **automatically** use the correct queue, so the thread you call this from does not matter. Keep in mind that the thread you call this from doesn’t matter, but you still have to abide by the rules of the NSManagedObject not being _thread-safe_. The NSMainQueueConcurrencyType is for use on the main queue, which is used by the _main thread_.

## Parent/Child Contexts

Back to parent and child contexts. These allow you to have a NSManagedObjectContext belong to another NSManagedObjectContext, sharing its persistentStoreCoordinator and thus persistentStore. This my friends is where the fun begins _(if you are nerdy and thing Core Data is fun like me)_.

Setting a NSManagedObjectContext instance as a child context of another instance is as easy as assigning its parentContext property to the context you wish to be the parent. The beauty of setting the context as a child is that **it doesn’t persist its changes to disk**. Instead it pushes its changes up to its parent but still triggers a NSManagedObjectContextDidSaveNotification. You can observe this notification and perform a **save on the parentContext writing the changes to disk**.

The benefit of this setup comes when you use a specific arrangement for the parent and the child contexts. We have a single parent context of type `NSPrivateQueueConcurrencyType` and this context is associated with the persistent store coordinator, we call this property the `privateQueueManagedObjectContext`. We keep this context private to the CoreDataController class since it is more of an implementation detail. We then have one public `NSMainQueueManagedObjectContext` property named `mainQueueManagedObjectContext` that is a **child** of our `privateQueueManagedObjectContext`, all changes from the `mainQueueManagedObjectContext` are pushed up to the `privateQueueManagedObjectContext` and **written to disk on the private queue using a background thread**. This is hugely beneficial by allowing you to save the `mainQueueManagedObjectContext` on the main thread and having it be very efficient since it is simply pushing its changes to the `privateQueueManagedObjectContext` and not actually writing to disk! Let’s take a look at the code to get this setup:

```objective-c
- (instancetype)init {
  NSPersistentStoreCoordinator *psc = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:self.managedObjectModel];
  NSError *error = nil;
  NSPersistentStore *store = [psc addPersistentStoreWithType:self.storeType configuration:self.modelConfiguration URL:self.storeURL options:self.storeOptions error:&error];

  NSAssert(store, @"error creating the core data stack, perhaps the managed object model changed?");
  if(store == nil) {
    NSLog(@"error creating core data stack: %@", error);
    return nil;
  }

  _privateQueueManagedObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
  _privateQueueManagedObjectContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
  _privateQueueManagedObjectContext.persistentStoreCoordinator = psc;

 [_privateQueueManagedObjectContext performBlock:^{
    [[_privateQueueManagedObjectContext userInfo] setValue:@"Private Queue" forKey:@"mocIdentifier"];
  }];

 _mainQueueManagedObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
  _mainQueueManagedObjectContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
  _mainQueueManagedObjectContext.parentContext = _privateQueueManagedObjectContext;
  [[_mainQueueManagedObjectContext userInfo] setValue:@"Main Thread" forKey:@"mocIdentifier"];

  NSNotificationCenter *defaultCenter = [NSNotificationCenter defaultCenter];
  [defaultCenter addObserver:self selector:@selector(handleManagedObjectContextDidSaveNotification:) name:NSManagedObjectContextDidSaveNotification object:nil];
  [defaultCenter addObserver:self selector:@selector(saveMasterContext) name:NSManagedObjectContextDidSaveNotification object:_mainQueueManagedObjectContext];
}
```

When the `mainQueueManagedObjectContext` fires a NSManagedObjectContextDidSaveNotification it will call `saveMasterContext` which essentially just calls save on the `privateQueueManagedObjectContext` and persists the changes to disk. We will take a look at the `handleManagedObjectContextDidSaveNotification:` method called by the observer in a moment but first I want to talk about _worker contexts_.

## Worker Contexts

We use _worker contexts_ to perform all sorts of operations in the background. Their lives can be short and that is fine since we only need them temporarily. However, we do need to know about the changes that they made so we can update our `privateQueueManagedObjectContext`. We created a helper method to create a new worker context

```objective-c
- (NSManagedObjectContext *)backgroundContext
{
  NSManagedObjectContext *context = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
  context.parentContext = self.privateQueueManagedObjectContext;
  context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy;
  [context performBlock:^{
    [[context userInfo] setValue:@"Child Context" forKey:@"mocIdentifier"];
  }];

  [self.contextDirectory addObject:context]; return context;
}
```

We don’t want to add ourselves as an observer of this context’s save notification since we will never be able to remove ourself as the observer. As you can see the way we initialize the background context is similar to the `mainQueueManagedObjectContext` except that this is also of the type `NSPrivateQueueConcurrencyType`. This allows us to have multiple contexts working in the background on different tasks such as importing data or asynchronously inserting data fetched from our API. You may notice we are adding this context to the _contextDirectory_. The _contextDirectory_ is a `NSHashTable`, a collection that holds weak references to the objects added to it. We use this _contextDirectory_ to decide if we should merge the changes into the `mainQueueManagedObjectContext` by ensuring the context triggering the notification exists in the directory.

The same rules apply when this context saves, its changes are pushed into the `privateQueueManagedObjectContext`. These however are not immediately written to disk, so how does it make it there?

## Writing Changes to Disk

Now we can take a look at the `handleManagedObjectContextDidSaveNotification:` method that is called whenever a `NSManagedObjectContextDidSaveNotification` is fired on any object.

```objective-c
- (void)handleManagedObjectContextDidSaveNotification:(NSNotification *)notification { NSManagedObjectContext *context = [notification object]; if (![self.contextDirectory containsObject:context] || context == self.mainQueueManagedObjectContext) { return; } [self.mainQueueManagedObjectContext performBlockAndWait:^{ [self.mainQueueManagedObjectContext mergeChangesFromContextDidSaveNotification:notification]; }]; [self saveMasterContext]; }
```

As you can see this method ensures that the context that triggered the notification is included in the _contextDirectory_ and that it isn’t the `mainQueueManagedObjectContext`. If it passes the initial checks then it merges in the changes to the `mainQueueManagedObjectContexts` inside a `performBlockAndWait` call, ensuring it happens on the correct queue. Once this finishes we go ahead and call `saveMasterContext` which in fact persists the changes to disk using the `privateQueueManagedObjectContext`.

Just so you can see the full implementation here is the saveMasterContext method:

```objective-c
- (void)saveMasterContext {
  UIBackgroundTaskIdentifier identifier = [[UIApplication sharedApplication] beginBackgroundTaskWithName:@"background-moc-save" expirationHandler:^{
    if([self.delegate respondsToSelector:@selector(coreDataController:didSaveWithError:)])
    {
      NSError *error = [NSError errorWithDomain:@"com.modi" code:3009 userInfo:@{NSLocalizedDescriptionKey : @"Failed to save before background task expired"}];
      [self.delegate coreDataController:self didSaveWithError:error];
    }
  }];

  [self.privateQueueManagedObjectContext performBlockAndWait:^{
    if ([self.privateQueueManagedObjectContext hasChanges])
    {
      NSError *error = nil; [self.privateQueueManagedObjectContext save:&error];
      if([self.delegate respondsToSelector:@selector(coreDataController:didSaveWithError:)])      { [self.delegate coreDataController:self didSaveWithError:error];
      }

      NSLog(@"saved background context");
    }

    [[UIApplication sharedApplication] endBackgroundTask:identifier];
  }];
}
```

We begin by starting a background task, this is because we call this method when we receive a notification that the application is going to enter the background. We then call `performBlockAndWait:` with a block that checks to see if the `privateQueueManagedObjectContext` has changes and if so triggering a save and notifying our delegate if necessary. After everything is saved and done, we end the background task.

All of the code above I placed in a singleton class named CoreDataController that exposes helper methods to create worker contexts, get the `mainQueueManagedObjectContext` and that is about it. All of the nitty gritty details are hidden away inside of the class without a need to be known by outsiders.

Do you have other examples of successful Core Data stacks? I would love to take a look at what you have.
