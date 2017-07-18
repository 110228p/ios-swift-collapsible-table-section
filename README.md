# How to Implement Collapsible Table Section in iOS

A simple iOS swift project demonstrates how to implement collapsible table section programmatically, that is no main storyboard, no XIB, no need to register nib, just pure Swift!

In this project, the table view automatically resizes the height of the rows to fit the content in each cell, and the custom cell is also implemented programmatically.

![rocket](docs/images/cover.gif)

[![Language](https://img.shields.io/badge/swift-3.0-brightgreen.svg?style=flat)]()

### How to implement collapsible table sections? ###

#### Step 1. Prepare the Data ####

Let's say we have the following data that is grouped into different sections, each section is represented by a `Section` object:

```swift
struct Section {
  var name: String
  var items: [String]
  var collapsed: Bool
    
  init(name: String, items: [Item], collapsed: Bool = false) {
    self.name = name
    self.items = items
    self.collapsed = collapsed
  }
}

struct Item {
  var name: String
  var detail: String
    
  init(name: String, detail: String) {
    self.name = name
    self.detail = detail
  }
}
    
var sections = [Section]()

sections = [
  Section(name: "Mac", items: [
    Item(name: "MacBook", detail: "Apple's ultraportable laptop"),
    Item(name: "MacBook Air", detail: "A very light ultraportable laptop.")
  ]),
  Section(name: "iPad", items: [
    Item(name: "iPad Pro", detail: "iPad Pro delivers epic power."),
    Item(name: "iPad Air 2", detail: "The second-generation of iPad Air tablet.")
  ]),
  Section(name: "iPhone", items: [
    Item(name: "iPhone 7", detail: "The latest iPhone."),
    Item(name: "iPhone 6", detail: "The 6th-generation of iPhone.")
  ])
]
```
`collapsed` indicates whether the current section is collapsed or not, by default is `false`.

#### Step 2. Setup TableView to Support Autosizing ####

```swift
override func viewDidLoad() {
  super.viewDidLoad()
        
  // Auto resizing the height of the cell
  tableView.estimatedRowHeight = 44.0
  tableView.rowHeight = UITableViewAutomaticDimension
  
  ...
}
```

Ignore this step if your cells have a fixed height.

#### Step 3. The Section Header ####

According to [Apple API reference](https://developer.apple.com/reference/uikit/uitableviewheaderfooterview), we should use `UITableViewHeaderFooterView`. Let's subclass it and implement the section header `CollapsibleTableViewHeader`:

```swift
class CollapsibleTableViewHeader: UITableViewHeaderFooterView {
    let titleLabel = UILabel()
    let arrowLabel = UILabel()
    
    override init(reuseIdentifier: String?) {
        super.init(reuseIdentifier: reuseIdentifier)
        
        contentView.addSubview(titleLabel)
        contentView.addSubview(arrowLabel)
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```

We need to collapse or expand the section when user taps on the header, to achieve this, let's borrow `UITapGestureRecognizer`. Also we need to delegate this event to the table view to update the `collapsed` property.

```swift
protocol CollapsibleTableViewHeaderDelegate {
    func toggleSection(_ header: CollapsibleTableViewHeader, section: Int)
}

class CollapsibleTableViewHeader: UITableViewHeaderFooterView {
    var delegate: CollapsibleTableViewHeaderDelegate?
    var section: Int = 0
    ...
    override init(reuseIdentifier: String?) {
        super.init(reuseIdentifier: reuseIdentifier)
        ...
        addGestureRecognizer(UITapGestureRecognizer(target: self, action: #selector(CollapsibleTableViewHeader.tapHeader(_:))))
    }
    ...
    func tapHeader(_ gestureRecognizer: UITapGestureRecognizer) {
        guard let cell = gestureRecognizer.view as? CollapsibleTableViewHeader else {
            return
        }
        delegate?.toggleSection(self, section: cell.section)
    }
    
    func setCollapsed(_ collapsed: Bool) {
        // Animate the arrow rotation (see Extensions.swf)
        arrowLabel.rotate(collapsed ? 0.0 : .pi / 2)
    }
}
```

Since we are not using any storyboard or XIB, how to do auto layout programmatically? The answer is `NSLayoutConstraint`'s `constraintsWithVisualFormat` function.

```swift
override init(reuseIdentifier: String?) {
    ...
    // arrowLabel must have fixed width and height
    arrowLabel.widthAnchor.constraintEqualToConstant(12).active = true
    
    titleLabel.translatesAutoresizingMaskIntoConstraints = false
    arrowLabel.translatesAutoresizingMaskIntoConstraints = false
}

override func layoutSubviews() {
    super.layoutSubviews()
    ...
    let views = [
        "titleLabel" : titleLabel,
        "arrowLabel" : arrowLabel,
    ]

    contentView.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat(
        "H:|-20-[titleLabel]-[arrowLabel]-20-|",
        options: [],
        metrics: nil,
        views: views
    ))

    contentView.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat(
        "V:|-[titleLabel]-|",
        options: [],
        metrics: nil,
        views: views
    ))

    contentView.addConstraints(NSLayoutConstraint.constraintsWithVisualFormat(
        "V:|-[arrowLabel]-|",
        options: [],
        metrics: nil,
        views: views
    ))
}
```

#### Step 4. The UITableView DataSource and Delegate ####

Now we implemented the header view, let's get back to the table view controller.

The number of sections is `sections.count`:

```swift
override func numberOfSectionsInTableView(in tableView: UITableView) -> Int {
  return sections.count
}
```

and the number of rows in each section is:

```swift
override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return sections[section].collapsed ? 0 : sections[section].items.count
}
```

Noticed that we don't need to render any cell for the collapsed section, this can improve the performance a lot if there are lots of cells in that section.

Next, we use tableView's viewForHeaderInSection function to hook up our custom header:

```swift
override func tableView(_ tableView: UITableView, viewForHeaderInSection section: Int) -> UIView? {
    let header = tableView.dequeueReusableHeaderFooterViewWithIdentifier("header") as? CollapsibleTableViewHeader ?? CollapsibleTableViewHeader(reuseIdentifier: "header")

    header.titleLabel.text = sections[section].name
    header.arrowLabel.text = ">"
    header.setCollapsed(sections[section].collapsed)

    header.section = section
    header.delegate = self

    return header
}
```

Setup the normal row cell is pretty straightforward:

```swift
override func tableView(_ tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {
  let cell: CollapsibleTableViewCell = tableView.dequeueReusableCell(withIdentifier: "cell") as? CollapsibleTableViewCell ?? CollapsibleTableViewCell(style: .default, reuseIdentifier: "cell")
        
  let item: Item = sections[(indexPath as NSIndexPath).section].items[(indexPath as NSIndexPath).row]
        
  cell.nameLabel.text = item.name
  cell.detailLabel.text = item.detail
        
  return cell
}
```

Of course you can use a plain `UITableViewCell`, our `CollapsibleTableViewCell` is a subclass of `UITableViewCell` that adds the name and detail labels, and the most important thing is that it supports autosizing feature, the key is to setup the autolayout constrains properly, please refer to the source code for more details.

#### Step 5. How to Toggle Collapse and Expand ####

The idea is really simple, if a section's `collapsed` property is `true`, we set the height of the rows inside that section to be `0`, otherwise `UITableViewAutomaticDimension`!

```swift
override func tableView(_ tableView: UITableView, heightForRowAtIndexPath indexPath: NSIndexPath) -> CGFloat {
  return sections[(indexPath as NSIndexPath).section].collapsed ? 0 : UITableViewAutomaticDimension
}
```

And here is the toggle function:

```swift
extension CollapsibleTableViewController: CollapsibleTableViewHeaderDelegate {
  func toggleSection(_ header: CollapsibleTableViewHeader, section: Int) {
    let collapsed = !sections[section].collapsed
        
    // Toggle collapse
    sections[section].collapsed = collapsed
    header.setCollapsed(collapsed)
    
    // Reload the whole section
    tableView.reloadSections(NSIndexSet(index: section) as IndexSet, with: .automatic)
  }
}
```

After the sections get reloaded, the number of cells in that section will be recalculated and redrawn. 

That's it, please refer to the source code and see the detailed implementation.

### More Collapsible Demo ###

Sometimes you might want to implement the collapsible cells in a grouped-style table, I have a separate demo at [https://github.com/jeantimex/ios-swift-collapsible-table-section-in-grouped-section](https://github.com/jeantimex/ios-swift-collapsible-table-section-in-grouped-section). The implementation is pretty much the same but slightly different.

### License ###

This project is licensed under the MIT license, Copyright (c) 2017 Yong Su. For more information see `LICENSE.md`.

Author: [Yong Su](https://github.com/jeantimex) @ Box Inc.
