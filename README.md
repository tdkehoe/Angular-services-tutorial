# Angular Services Tutorial

Services can transfer values between components. Services can also simplify functions by moving async calls or logic to a service. Other functions can then call the service instead of repeating code. The result is short, easy to read functions in your components.

## Setting up a service

I put my services in a folder at `src/app/services`.

I make one service for each view. All the components of the view then use that service.

Alternatively you could make one service for each module, and keep the service in the module.

Just a few special items are needed in a service. First, import `Injectable` from `@angular/core`, import `Observable` and `BehaviorSubject` from `rxjs`, and make the service injectable:

*my-service.service.ts
```js
import { Injectable } from '@angular/core';
import { Observable, BehaviorSubject } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class MyService {

}
```

### No `OnInit`

Lifecycle hooks work with components and directives. They do not work with services. This is because components have a lifecycle when services don't have lifecycles. The eight lifecycle hooks are (in order of lifecycle execution):

```
ngOnChanges()
ngOnInit()
ngDoCheck()
ngAfterContentInit()
ngAfterContentChecked()
ngAfterViewInit()
ngAfterViewChecked()
ngOnDestroy()
```

The exception is `ngOnDestroy()`. Services can be destroyed and `ngOnDestroy()` can be used in services.

### Constructor

You can put stuff in a constructor to execute when a service is called. This code imports Firestore and AngularFire, imports another service, and then gets the logged in `userID` from the other service.

*my-service.service.ts
```js
import { Injectable } from '@angular/core';
import { Observable, BehaviorSubject } from 'rxjs';

// AngularFire
import { Firestore, doc, DocumentSnapshot, getDoc, getDocs, collection, setDoc, updateDoc } from '@angular/fire/firestore';
import { User } from '@angular/fire/auth';

// services
import { HeaderToolbarService } from 'src/app/services/header-toolbar/header-toolbar.service';

@Injectable({
  providedIn: 'root'
})
export class MyService {
 docSnap: DocumentSnapshot;
 user: User;
  
 constructor(
    public firestore: Firestore,
    @Inject(HeaderToolbarService) private headerToolbarService: HeaderToolbarService,
  ) {
    // get user
    this.headerToolbarService.getUser().subscribe((userObject: any) => {
      this.user = userObject;
    });
  }
  
}
```

### Making types

This isn't required but a service is a good place to put your types.

*my-service.service.ts
```js
import { Injectable } from '@angular/core';
import { Observable, BehaviorSubject } from 'rxjs';

// AngularFire
import { Firestore, doc, DocumentSnapshot, getDoc, getDocs, collection, setDoc, updateDoc } from '@angular/fire/firestore';
import { User } from '@angular/fire/auth';

// services
import { HeaderToolbarService } from 'src/app/services/header-toolbar/header-toolbar.service';

export declare type LearningModes = 'videos' | 'vocabulary' | 'wordSearch';

export declare type VideoSource = {
  src: string;
  type: string;
};

@Injectable({
  providedIn: 'root'
})
export class MyService {
 docSnap: DocumentSnapshot;
 user: User;
  
 constructor(
    public firestore: Firestore,
    @Inject(HeaderToolbarService) private headerToolbarService: HeaderToolbarService,
  ) {
    // get user
    this.headerToolbarService.getUser().subscribe((userObject: any) => {
      this.user = userObject;
    });
  }
  
}
```

## Transfer values with `set` and `get`

Now we're ready to make some service functions. 

If a value is used by more than one component it should be on a service. This is the easiest service function or, rather, pair of functions. One functions sets the value, the other gets the value. This code shows only the class `MyService`:

*my-service.service.ts
```js
export class MyService {
 docSnap: DocumentSnapshot;
 user: User;
  
 constructor(
    public firestore: Firestore,
    @Inject(HeaderToolbarService) private headerToolbarService: HeaderToolbarService,
  ) {
    // get user
    this.headerToolbarService.getUser().subscribe((userObject: any) => {
      this.user = userObject;
    });
  }
  
  // executes when user selects video in video-vocabulary.component
  videoTitle: VideoTitle | null = null;
  videoTitleObservable: BehaviorSubject<VideoTitle | null> = new BehaviorSubject(this.videoTitle);
  setVideoTitle(title: VideoTitle): void {
    this.videoTitleObservable.next(title);
  }
  getVideoTitle(): Observable<VideoTitle | null> {
    return this.videoTitleObservable.asObservable();
  }
  
}
```

The service function declares local variables. The first variable, `videoTitle`, holds the value to be transferred between components. The second variable, `videoTitleObservable` is special for services. This is an Observable and makes the value available when the service is called. Follow this pattern for any "getter/setter" service function.

### `return`

Your `get` service function must return something.

### Comments

Note that I put in a comment saying which component sets the value, and which component gets the value.

### Accessing a `set` service from a component
Setting a value from a component can be done anywhere in your component, usually after a value has changed.

*my-component.ts*
```js
this.myService.setVideoTitle(this.videoTitle); // send sentence to home-toolbar service, then speech-to-text.component.ts
```

### Accessing a `get` service from a component

A component usually gets a value by setting up a listener in `ngOnInit()`, in other words, at the beginning of the component lifecycle.

*my-component.ts*
```js
// listen for the user selecting a new video
this.MyService.getVideoTitle().subscribe((videoTitle: VideoTitle) => {
   this.videoTitle = videoTitle;
});
```

The local variable `this.videoTitle` will then always have the latest value.

## Async services

If several components access a value from the same location in a database, make this a service. This service sets a value in the database and then gets the value from the database.

*my-service-service.ts*
```js
sentencePronounced: boolean = false;
sentencePronouncedObservable: BehaviorSubject<boolean> = new BehaviorSubject(this.sentencePronounced);
async setSentencePronounced(sentencePronounced: boolean): Promise<void> {
    this.getVideoTitle().subscribe((videoTitle: VideoTitle | null) => {
      this.videoTitle = videoTitle;
    });
    this.getClipInMovieWatching().subscribe((clip: number | undefined ) => {
      this.clipInMovieWatching = clip;
    });
    const usersClipRef = doc(this.firestore, 'Users/' + this.user.uid + '/' + this.l2Language.long + '/Videos/' + this.videoTitle?.snake_title, this.videoTitle?.snake_title + '_' + this.clipInMovieWatching);
    await setDoc(usersClipRef, { sentencePronounced: sentencePronounced }, { merge: true }); // updateDoc would work too
    this.sentencePronouncedObservable.next(sentencePronounced);
}
async getSentencePronounced(): Promise<Observable<boolean>> {
    const usersClipRef = doc(this.firestore, 'Users/' + this.user.uid + '/' + this.l2Language.long + '/Videos/' + this.videoTitle?.snake_title, this.videoTitle?.snake_title + '_' + this.clipInMovieWatching);
    this.docSnap = await getDoc(usersClipRef); // query Firebase for the sentencePronounced
    console.log(this.docSnap.data()?.['sentencePronounced']);
    if (this.docSnap.data()?.['sentencePronounced'] === undefined) {
      await setDoc(usersClipRef, { sentencePronounced: false }, { merge: true }); // updateDoc would work too
      this.sentencePronouncedObservable.next(this.sentencePronounced);
      return this.sentencePronouncedObservable.asObservable();
    } else {
      this.sentencePronounced = this.docSnap.data()?.['sentencePronounced'];
      this.sentencePronouncedObservable.next(this.sentencePronounced);
      return this.sentencePronouncedObservable.asObservable();
    }
}
```

We put `async` in front of each service function.

The `get` service function has two returns, one synchronous and one async.

### Accessing an async `get` service from a component

Setting a value in an async function is the same as with a synchronous function.

To get a value from an async service function:

```js
async callMe() {
    (await
      this.homeToolbarService.whereAreWeInWordList()).subscribe((state: string) => {
      console.log(state);
    });
}
```

The calling function is async. The Observer is preceded by `await` and enclosed in self-executing parantheses.

### Async set and get

Let's send a value to a setter service function, access the database, and return the data to the caller.

*my-component.ts*
```js
this.homeToolbarService.setPronunciations('kitten');
  (await
      this.homeToolbarService.getPronunciations()).subscribe((array: Pronunciation[]) => {
        console.table(array);
    });
```

We pass a value to the setter synchronously, then make an async Observor to wait for the response from the getter.

*my-service.service.ts*
```js
  pronunciations: Pronunciation[] = [];
  pronunciationsObservable: BehaviorSubject<Pronunciation[]> = new BehaviorSubject(this.pronunciations);
  async setPronunciations(word: string): Promise<void> {
    this.pronunciations = [];
    const pronunciationsRef = 'Dictionaries/' + this.l2Language.long + '/Words/' + word + '/Pronunciations/';
    this.querySnapshot = await getDocs(collection(this.firestore, pronunciationsRef));
    this.querySnapshot.forEach((document: any) => {
      console.table(document.data());
      this.pronunciations.push(document.data());
    });
    this.pronunciationsObservable.next(this.pronunciations);
  }
  async getPronunciations(): Promise<Observable<Pronunciation[]>> {
    return this.pronunciationsObservable.asObservable();
  }
```

We call the database asynchronously to get a collection, iterate through the collection to make an array, and pass the array to the Observable getter service function.
