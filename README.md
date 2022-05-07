# angular-auth

to generate new angular application.

```bash
ng new <app-name> --routing=true --style=scss
```

add "HttpClientModule" module to module.ts file to call api.

```typescript
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http'; // add +

@NgModule({
  declarations: [
    AppComponent,
  ],
  imports: [
    BrowserModule,
    AppRoutingModule,
    BrowserAnimationsModule,
    ReactiveFormsModule, // for form builder validation
    FormsModule, // for forms bind
    HttpClientModule, // to call api from frontend  ++ add +

  ],
  providers: [],
  bootstrap: [AppComponent]
})
```

to generate service.

```bash
ng g s  </services/auth/authentication>
```

and add login/register code for authentication.

```typescript
// add all
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from 'src/environments/environment';
import { Router } from '@angular/router';


@Injectable({
  providedIn: 'root'
})
export class AuthenticationService {

  constructor(private http: HttpClient,
              private router: Router) { }


  userLogin(userLogin:any){
      console.log(userLogin);
      return this.http.post(environment.clientApi+'/client/login',
      {
        userName: userLogin.username,
        passWord: userLogin.password
      });
    }

  userRegister(userRegister:any){
      console.log(userRegister);
      return this.http.post(environment.clientApi+'/client/register',
      {
        userName: userRegister.username,
        passWord: userRegister.password
      });
  }

  loggedIn(){
    return !!localStorage.getItem('token');
  }

  userLogout(){
     localStorage.removeItem('token');
     this.router.navigate(['/']);
  }

  getToken(){
    return localStorage.getItem('token');
  }

  getCounter(){
    return localStorage.getItem('counter');
  }

}
```

add "AuthenticationService" tp app.module.ts file.

```typescript
import { AuthenticationService } from 'src/app/services/auth/authentication.service'; // add +


@NgModule({
  declarations: [
    AppComponent,
  ],
  imports: [
    BrowserModule,
    AppRoutingModule,
    FormsModule,
    ReactiveFormsModule,
    NgbModule
  ],
  providers: [AuthenticationService], // add +
  bootstrap: [AppComponent]
})
```

use "AuthenticationService" file on login/register.     
1.store jwt token in "localStorage"    
2.if login/register success then route.     

>register.component.ts

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';

// from validation
import { FormGroup, FormBuilder, Validators  } from '@angular/forms';

// serviece
import { AuthenticationService } from 'src/app/services/auth/authentication.service';

@Component({
  selector: 'app-register',
  templateUrl: './register.component.html',
  styleUrls: ['./register.component.scss']
})
export class RegisterComponent implements OnInit {

  userRegisterForm:FormGroup;

  constructor(
    private fb:FormBuilder,
    private auth: AuthenticationService,
    private router:Router,
  ) { }

  ngOnInit(): void {
    this.validateRegisterForm();
  }

  validateRegisterForm():void{
    this.userRegisterForm = this.fb.group({
      userNickName:['',[Validators.required]], // onlyNumbers(),onlyIntegers()
      userEmail:['',[Validators.required,Validators.email]],
      userCountry:[''],
      userPassword:['',[Validators.required]],
    });
  }

  isValidedForm():boolean{
    this.userRegisterForm.markAllAsTouched();
    return this.userRegisterForm.valid;
  }


  register():void{
    // console.log("click register", this.userRegisterForm.value);
     this.auth.userRegister(this.userRegisterForm.value).subscribe(res=>{
      if(res && res['success']){
        localStorage.setItem('token', res['token'] );
        this.router.navigate(['/']);
        // this._toastr.success('successfully load customer', 'Success');
        console.log("register success !!");
      }else{
        //this.toastr.error('user credentials not matched', 'Error');
        const info = res['data']
        console.log("register problem",info);
        // this._toastr.info(info, 'Info');
      }

    },err=>{
      //this.toastr.error(err, 'Error');
      // this._toastr.error( err.message, 'Error');
      console.log("register err",err);
    });
  }

}
```

to create Route Guard
```bash
$ ng g guard /services/auth/auth
```  

Route Guard code    

```typescript  
// +add all the code

import { Injectable } from '@angular/core';
// import { CanActivate, ActivatedRouteSnapshot, RouterStateSnapshot, UrlTree, Router } from '@angular/router';
// import { Observable } from 'rxjs';
import { CanActivate, Router } from '@angular/router';
import { AuthenticationService } from 'src/app/services/auth/authentication.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {

  // canActivate(
  //   next: ActivatedRouteSnapshot,
  //   state: RouterStateSnapshot): Observable<boolean | UrlTree> | Promise<boolean | UrlTree> | boolean | UrlTree {
  //   return true;
  // }

  constructor(private auth:AuthenticationService,
              private router:Router){}

  canActivate():boolean{
    if(this.auth.loggedIn()){
      return true;
    }else{
      this.router.navigate(['/login']);
      return false;
    }
  }            

}
```

add above "Route Guard" to "app-routing.module.ts" file.
1.import route guard .

> app-routing.module.ts   

```typescript
import { AuthdGuard } from 'src/app/services/auth/guard.guard'; // +add

const routes: Routes = [
  {
    path: '',
    pathMatch: 'full',
    component: TemplateComponent
  },
  {
    path: 'main',
    component: MainComponent,
    canActivate:[AuthdGuard], // +add
  }

];
```

add token "INTERCEPTORS"
1.create new service for token interceptor.    

```bash
$ ng g s /services/auth/token-interceptor
```

2.handle interceptor token     
```typescript
// import { Injectable } from '@angular/core';
import { Injectable , Injector} from '@angular/core'; // +add for issue
import { HttpInterceptor } from '@angular/common/http'; // +add
import { AuthenticationService } from 'src/app/services/auth/authentication.service'; // +add for issue

@Injectable({
  providedIn: 'root'
})
export class TokenInterceptorService implements HttpInterceptor{

  constructor(private injector:Injector) { } // +add for issue

  intercept(req,next){
    let authSerivce = this.injector.get(AuthenticationService); // +add for issue
    let tokenizedReq = req.clone({
      setHeader:{
        Authorization: `Bearer ${authSerivce.getToken()}`
      }
    });
    return next.handle(tokenizedReq);
  }
}
```

3.register interceptor service in "app.module.ts" file.   

> app.module.ts

```typescript
import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http'; // + add


providers: [AuthenticationService,
{                                      // +add
   provide:HTTP_INTERCEPTORS,          // +add
   useClass:TokenInterceptorService,   // +add
   multi:true                          // +add
}                                      // +add
],

```



add logout function.      

1.just import "AuthenticationService".    
2.call logout function    
