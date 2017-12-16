## M앱 마이그레이션 구현
참고 샘플 : **`sample_main`**

### 1. 기존의 버즈스크린 연동은 변경하지 않습니다.

### 2. 버즈스크린 SDK 업데이트 확인
마이그레이션은 버즈스크린 SDK 버전 1.6.0 이상에서 지원합니다. `build.gradle`에서 버즈스크린 버전이 `1.+` 인 경우는 자동으로 업데이트되기 때문에 추가작업이 필요없지만, 특정 버전으로 지정된 경우는 반드시 1.6.0 버전 이상을 사용해야 합니다.

아래와 같이 버즈스크린 라이브러리 추가 코드 확인.
```
dependencies {
    compile 'com.buzzvil:buzzscreen:1.+'
}

```

### 3. `build.gradle` 설정

#### `manifestPlaceholders` 에 추가

```
android {
    defaultConfig {
        // my_app_key 에는 버즈스크린 연동시 발급받은 앱키를 입력합니다.
        manifestPlaceholders = [buzzScreenAppKey:"my_app_key"]
    }
}
```

#### `dependencies` 에 추가
**L앱과 설정이 다름에 주의**합니다.

```
dependencies {
    compile 'com.buzzvil:buzzscreen-migration-from:1.+'
}
```

### 4. Application Class 에 코드 추가
- `MigrationFrom.init(Context context, String lockScreenPackageName)`

    마이그레이션을 위한 M앱의 초기화 코드

    **Parameters**
    - `context` : Application context 를 `this` 로 전달
    -  `lockScreenPackageName` : L앱의 패키지명

- `MigrationFrom.bind(OnMigrationListener onMigrationListener)`

    마이그레이션 진행을 위한 함수입니다. 마이그레이션 과정을 `onMigrationListener`를 통해 수신받을 수 있고, 마이그레이션시에 M앱에서 L앱으로 옮기고 싶은 정보를 전달할 수 있습니다. **반드시 `MigrationFrom.init` 후에 호출**합니다.

    **Parameters**
    - `onMigrationListener`
        - `onAlreadyMigrated` : 이미 마이그레이션이 진행된경우 호출됩니다.
        - `onMigrationStarted` : 마이그레이션 시작시 호출됩니다. 리턴값을 통해 M앱에서 L앱으로의 정보 전달이 가능합니다. M앱의 유저인증정보를 전달하여 L앱에서 추가적인 유저 액션 없이 자동로그인을 구현할 수 있습니다.
        - `onMigrationCompleted` : 마이그레이션 종료시 호출됩니다. 마이그레이션 종료시 M앱에서 버즈스크린이 자동으로 off 됩니다. 만약 on/off 정보를 기록하고 있다면 여기서 off 상태로 변경바랍니다.

**사용 예시**

```Java
public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        // Do not touch. 기존 버즈스크린 초기화 코드.
        BuzzScreen.init("app_key", this, CustomLockerActivity.class, R.drawable.image_on_fail);

        // 마이그레이션을 위한 코드
        // L앱의 패키지명이 com.buzzvil.sample_lock 인 경우 사용 예시
        MigrationFrom.init(this, "\"com.buzzvil.sample_lock\"");
        MigrationFrom.bind(new MigrationFrom.OnMigrationListener() {
            @Override
            public void onAlreadyMigrated() {
            }

            @Override
            public Bundle onMigrationStarted() {
                // M앱에서 이미 설정한 버즈스크린 UserProfile, 잠금화면 on/off 정보는 자동으로 L앱으로 전달됩니다.
                // 그 외의 정보 전달을 리턴해주면 됩니다.
                // 반드시 유저 인증정보를 전달하여 L앱에서 유저의 추가 액션없이 자동 로그인이 되도록 구현바랍니다.
                // 여기서는 user_id 를 유저인증정보로 가정하고 전달하여 L앱에서 자동로그인을 구현하였습니다.
                if (isLoggedIn()) {
                    Bundle bundle = new Bundle();
                    bundle.putString("user_id", PreferenceHelper.getString(Constants.PREF_KEY_USER_ID, ""));
                    return bundle;
                } else {
                    return null;
                }
            }

            @Override
            public void onMigrationCompleted() {
                // 마이그레이션 완료상태를 저장하고 싶은 경우 사용 예시
                PreferenceHelper.putBoolean(Constants.PREF_KEY_MIGRATION_COMPLETED, true);
            }
        });
    }
}

```