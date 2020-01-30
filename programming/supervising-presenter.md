# Supervising Presenter

These notes were taken from: https://martinfowler.com/eaaDev/SupervisingPresenter.html

- Similar to standard MVC model
- Controller both handles input response and manipulates view to handle complex view logic
- Simple view behavior is left to the declarative system
- The controller only intervenes in the view when necessary

## How it Works

- Responsibilities
  - Input response
  - Partial view/model synchronization
- Input response
  - Identical in behavior to a standard presenter
  - User gestures are initially handled by the widgets in the view
  - The view does nothing but forward information about the user's gestures to the presenter
  - The presenter handles all further logic
- View/model synchronization
  - Defer as much as possible to the view
  - Data binding should be used whenever possible
  - When data binding is not possible, the controller steps in

## Motivation

- Testability
  - If the view is hard to test then moving logic into the controller and using a test double can help
  - Another option is to use a gateway driver for the view and have a separate implementation for testing
- Separation of complexity from the view window itself
  - Controller is still tightly coupled to the view, so this may not be a huge win

## Questions possibly answered by reading more about the MVC pattern

- Are `actualField` and `varianceField` technical terms usced by the MVC pattern?

## Additional Reading

- https://martinfowler.com/eaaDev/DataBinding.html
- https://martinfowler.com/eaaCatalog/domainModel.html
- http://www.object-arts.com/papers/TwistingTheTriad.PDF
- http://xunitpatterns.com/Test%20Double.html
- https://martinfowler.com/eaaCatalog/gateway.html
- https://martinfowler.com/eaaDev/FlowSynchronization.html
- https://martinfowler.com/eaaDev/MediatedSynchronization.html
- https://martinfowler.com/eaaDev/AutonomousView.html
- https://martinfowler.com/eaaDev/PassiveScreen.html
- https://martinfowler.com/eaaDev/PresentationModel.html
