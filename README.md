1. Run `npm ci`
1. Run `npm run test`
1. Note how Chrome console has a warning from the Angular router "Navigation triggered outside Angular zone, did you forget to call 'ngZone.run()'?"
1. Run `git checkout angular13`
1. Run `npm ci`
1. Run `npm run test`
1. Note how the Chrome console no longer has the router warning

Also note this only happens when the Router is included as a dependency in the component under test; if this line is deleted the warning no longer appears: https://github.com/JoshuaEN/angular-14-router-unit-test-log-warning/blob/8d8497758851bf8ed33605a1257516ef50310fb5/src/app/app.component.ts#L11 

# Root Cause
Previously, the NgModuleRef constructor would call `_r3Injector._resolveInjectorDefTypes()`, which in turn ran the `Router`'s constructor. Now, `_r3Injector.resolveInjectorInitializers();` is called instead and the actual creation of the `Router` instance is deferred.

Specifically, it appears to be deferred to `initComponent` in `createComponent`, which is run inside of a zone with the `isAngularZone` property set to `true`: https://github.com/angular/angular/blob/8340da92bb75f62961d6cc99bc223a9138929688/packages/core/testing/src/test_bed.ts#L562-L567

This ultimately causes `router.isNgZoneEnabled` to evalulate to `true` (here: https://github.com/angular/angular/blob/8340da92bb75f62961d6cc99bc223a9138929688/packages/router/src/router.ts#L582 ), triggering the warning when calling navigate.

The reason not including the `Router` as a dependency of the component fixes the problem appears to be because the initialization of the `Router` service is deferred to the `TestBed.inject` call, which is not inside an Angular zone.
