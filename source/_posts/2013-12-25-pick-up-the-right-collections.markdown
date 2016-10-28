---
title: Pick up the right collections
date: 2013-12-25 17:53:30
---

代码1：

	NSArray *sessionsFromServer = ...;
	NSManagedObjectContext *context = ...;

	for (WWDCSession *aSession in sessionsFromServer) {
	    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"WWDCSession"];
	    req.predicate = [NSPredicate predicateWithFormat:@"sessionID == %@", aSession.sessionID];
	    NSArray *results = [context executeFetchRequest:req error:nil];

	    WWDCSession *existSesion = [results firstObject];
	    if (!existSesion) {
	        //Insert a New One
	    }else {
	        // Do something with the existSession
	    }
	}


代码2：

    NSArray *sessionsFromServer = ...;
    NSManagedObjectContext *context = ...;

    NSArray *sessionIDs = [sessionsFromServer valueForKey:@"SessionID"];
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"WWDCSession"];
    req.predicate = [NSPredicate predicateWithFormat:@"sessionID in %@", sessionIDs];
    NSArray *results = [context executeFetchRequest:req error:nil];

    for (WWDCSession *aSession in sessionsFromServer) {
        NSInteger existingSessionIndex = [results indexOfObject:aSession];
        if (existingSessionIndex == NSNotFound) {
            //Insert a New One
        }else {
            // Do something with the existSession
        }
    }

代码3：

    NSArray *sessionsFromServer = ...;
    NSManagedObjectContext *context = ...;

    NSArray *sessionIDs = [sessionsFromServer valueForKey:@"SessionID"];
    NSFetchRequest *req = [NSFetchRequest fetchRequestWithEntityName:@"WWDCSession"];
    req.predicate = [NSPredicate predicateWithFormat:@"sessionID in %@", sessionIDs];
    NSArray *results = [context executeFetchRequest:req error:nil];
    NSDictionary *mapDic = [NSDictionary dictionaryWithObjects:results forKeys:[results valueForkey:@"sessionID"]];

    for (WWDCSession *aSession in sessionsFromServer) {
        WWDCSession *existingSession = mapDic[aSession.sessionID];
        if (!existingSession) {
            //Insert a New One
        }else {
            // Do something with the existSession
        }
    }

代码1循环里每次都要进行次数据库查询。

代码2改进数据查询为一次，但是for循环里面还是for了一遍results。

代码3继承代码2的优点，但是for循环里面根据Dictionary的优势只进行了一次查询，可谓高手风范。

有些法则需要遵循：

1. 特别要注意循环里面干什么！
2. 可能的话，尽量使用NSDictionary和NSSet来查询。（这两个都是Hash table store）
3. 能使用mutable collections(或者strings)的时候就用他们来addObject吧。不要重新来初始化一个新的inmutable的了。
4. 精简数据库查询代码，比如代码1中的这种，能一次搞定就一次搞定。
