# Architectures

* [MVC](mvc.md)
* [MVP](mvp.md)
* [MVVM](mvvm.md)
* [VIP](vip.md)
* [VIPER](viper.md)

## Câu hỏi phỏng vấn tổng quan

**Q: MVC/MVP/MVVM khác ở UI update?** MVC controller điều phối trực tiếp; MVP presenter gọi passive view; MVVM expose observable state/binding.

**Q: VIP và VIPER khác gì?** VIP unidirectional ViewController→Interactor→Presenter→ViewController; VIPER module View/Interactor/Presenter/Entity/Router.

**Q: Clean Architecture có bắt buộc VIPER?** Không, Clean định hướng dependency vào domain, presentation có thể MVC/MVVM/VIPER.
