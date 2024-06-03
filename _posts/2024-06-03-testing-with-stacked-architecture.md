---
layout: post
title: Testing with Stacked Architecture
date: 2024-06-03 22:26 +0300
categories: [Flutter, Dart, Test]
tags: [all, important, test]
description: Selamlar selamlarrrr, normalde bu blog'u ingilizce yazacaktım fakat...
---
# Stacked mimarisi ile Flutter'da test yazımı

Selamlar selamlarrrr, normalde bu blog'u ingilizce yazacaktım fakat neden ingilizce diye sorduğumda bir neden bulamadım. O yüzden şu an türkçe içerik olarak profilime ekliyorum.

Buraya geldiğinize göre Flutter konusunda az çok bilgi sahibi olduğunuzu düşünüyorum :) Ya da işime öyle geliyor.

Şimdi öncelikle Stacked'ı gördüğünüzde zihninizde canlanan bir şey yoksa sizi devam etmeden başka uzaylara yönlendirmek istiyorum. Yoksa buralar ürkütücü olabilir…

Stacked için şuradan başlayabilirsiniz: https://stacked.filledstacks.com/docs/getting-started/overview

Güzel bu blog için temel gereksinimlerin birini varsayımla diğerini linkle çözdük diye düşünüyorum. Büyük ihtimalle işime böyle geldi yine neyse başlayalım.

Bir mobil uygulama geliştirirken en önemli şey nedir? Tabi ki test'inin yazılması dememi bekliyorsunuz ama demeyeceğim. Bunlar hep şov. Bence keyif almak önemli özellikle ekiple birlikte keyif almak önemli.

Şimdi ekip dedik iyi güzel bir ton geliştirme yapacağız bu kodlar nasıl olacak da bir ahenk içinde vücut bulacak? Tabi ki test :)

Test diyince zihninizde canlanan bir şey var mı diye sormayacağım artık yoksa bu blog sabrımın testi olabilir…
Test için öncelikle örnek bir servis yazalım mı? Mecbur yazacağız bu soruları niye soruyorum acaba.

### Servis (Api olanından)
```dart
class UserApiService {
  Future<UserModel?> fetchUser(String userName) async {
    return await Future.delayed(
      const Duration(seconds: 3),
      () {
        return UserModel(name: 'Actual Api Service', age: 99);
      },
    );
  }
}
```

Evet arkadaşlar bomba gibi bir servis ile karşınızdayım. Servisi özetleyecek olursak tek bir metodu var fetchUser adında. Bu fetchUser metodu senaryoyu gerçek kılmak için Future olarak nullable bir UserModel dönecek. Tabii önce parametre olarak bir userName alacak(Buna userID deseymişiz iyiymiş, aslında çok geç değil değişmek için ama değişmeyeceğim). Güzel basit bir api servisini kurguladık.

### ViewModel
```dart
class HomeViewModel extends BaseViewModel {
  ///* Api Service

  final UserApiService _userApiService = locator<UserApiService>();

  //* Prop
  UserModel? _userModel;

  UserModel? get userModel => _userModel;

  Future<void> init(String someData) async {
    _userModel = await _userApiService.fetchUser(someData);
    print('init:: ${_userModel?.name}');
  }
}
```

Şu locator<UserApiService>() var ya tam bir büyü gibi. Napıyoruz HomeViewModel'de? BaseViewModel den extend etmişiz (Dane' paşanın nimetlerinden yararlanacağız tabi bu örnekte pek yararlanmamışız). Ardından kullanacağımız api servisinin nesnesini oluşturmuşuz büyülü bir şekilde (final UserApiService _userApiService = locator<UserApiService>()). Son olarak UserModel'im var bir de init metodum var.

init metodum ne yapıyor someData olarak bir girdi alıyor ve bu girdi ile api isteğini gerçekleştiriyor. Dönen cevabı da viewModelde tutulan _userModel'e atıyor.

Şimdi güzel yerlere doğru yola çıkıyoruz. Ben bu viewModel'de aldım bir bağımlılığımı çat diye sınıfın içinde oluşturdum. Hangisi bu tabi ki _userApiService. Bir de güzel init içerisinde direkt bu bağımlılığımı kullandım. E ben bunu test ederken napacağım Mock yaratsam onu nasıl kullanacak? Kullanacak merak etmeyin :)

### Mock Service (Yine Api olanından)
```dart
MockUserApiService getAndRegisterUserApiService() {
  _removeRegistrationIfExists<UserApiService>();
  final service = MockUserApiService();
  locator.registerSingleton<UserApiService>(service);
  when(service.fetchUser(any)).thenAnswer(
    (realInvocation) => Future.value(UserModel(name: "muss", age: 2)),
  );
  return service;
}
```
Biz buralara girdik de mockito muhabbeti var bir de. Zor bu blog işleri… Mockito da bu bizim UserApiService' mocklamamızı sağlıyor neticede şu MockUserApiService generate ediliyor özetle. Hadi ona da bir link bırakayım.

[Mock class - mockito library - Dart API](https://pub.dev/documentation/mockito/latest/mockito/Mock-class.html)

Şimdi bu getAndRegisterUserApiService metodu yine Dane paşanın nimeti bu viewmodel ve servisler için standart alanları otomatik oluşturuyor. Biz de bunu kullanacağız şimdi. Burada register işlemlerine takılmayın singleton olması ile alakalı. Bizim ilgilendiren kısım tam şurası;
```dart
when(service.fetchUser(any)).thenAnswer(
    (realInvocation) => Future.value(UserModel(name: "muss", age: 2)),
  );
```
Napmışız hangi fetchUser metodu hangi parametreyle çağrılırsa çağrılsın şunu dön demişiz;
```dart
UserModel(name: "muss", age: 2)
```
Döner mi? Sanmam. (Kod ile ilgili değil kafanız karışmasın)

Şimdi gidelim viewModel testimizi yazalım bitirelim.

### ViewModel Test
```dart
void main() {
  HomeViewModel getModel() => HomeViewModel();

  group('HomeViewmodelTest -', () {
    setUp(() => registerServices());
    tearDown(() => locator.reset());
    group('User Test', () {
      test('Fetch user', () async {
        final HomeViewModel viewModel = HomeViewModel();
        await viewModel.init('userName');
        print(viewModel.userModel?.name);
        expect('muss', viewModel.userModel?.name);
      });
    });
  });
}
```
Burası tamamen Dane'in nimeti setUp ile bağımlıkları ayaklandırıyor. teardown ile uçuruyor. viewModel.init('userName') bize ne döndürür?
Asıl api servisinin cevabını mı?
```dart
UserModel(name: 'Actual Api Service', age: 99)
```
Mock api servisinin cevabını mı?
```dart
UserModel(name: "muss", age: 2)
```
Mock api servisinin cevabını dönecek ve expect'imiz kabul görecek. Yani viewModellerinizde api isteklerine bağımlı olan metodlarınızı özgürce istediğiniz parametreler ile test edebileceksiniz…

Buradaki kodların tamamı için: https://github.com/mustafayilmazdev/stacked_test

Ne kadar anlaşılır oldu bilmiyorum ama merak ettiğiniz bir şey varsa benimle iletişime geçebilirsiniz.