---
layout: default
title: WideStrings
abstract:
  Some ROS2 users would like to use multi-byte characters for topic data, suitable for supporting multi-byte languages.
  This article lays out the problem, and explores potential solutions to this problem.
author: '[Chris Lalancette](https://github.com/clalancette)'
published: true
---

- This will become a table of contents (this text will be scraped).
{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}

## Background

ROS2 currently supports the concept of strings, which are sequences of single-byte characters.
When working with ROS 2 topic data, some users would like to use multi-byte characters (e.g. UTF-8) for their topic data.
This article explores the problem and the possible solutions.
Note that this article specifically does not talk about binary data, as that is already handled by using uint8_t arrays.
It also does not talk about using multi-byte characters for the topic *names*, as this is disallowed by the DDS specification.

## Multi-byte character background

Before delving into ROS 2 specifics, some time should be spent discussing multi-byte characters in general.
What is meant by multi-byte characters?
The original ASCII character set only provided for printable characters using binary sequences 0 to 127.
Several extensions were made to this to use 128-255, but that approach only extended the character set to be able to cover a few more languages.
Several other encodings were created as the need arose, and it became clear by the 1990's that a worldwide standard was required.
Thus, Unicode was created to be a single specification that could cover all of the languages of the world.
The early versions of Unicode actually failed to live up to this expectation, and some of those early versions of Unicode are still baked into widely used software (Microsoft Windows, Java, etc).
Later on, the Unicode standard evolved to truly be able to cover the languages of the entire world, and this is what should generally be used for new designs.
As a side benefit, the newer versions of Unicode (UTF-8) are backwards compatible with ASCII.
There is a lot of history here, and things are much more complicated then the brief introduction above.
For more information, the following links provide introductory material:

* [http://kunststube.net/encoding/](http://kunststube.net/encoding/)
* [http://stackoverflow.com/questions/4588302/why-isnt-wchar-t-widely-used-in-code-for-linux-related-platforms](http://stackoverflow.com/questions/4588302/why-isnt-wchar-t-widely-used-in-code-for-linux-related-platforms)
* [http://www.diveintopython3.net/strings.html](http://www.diveintopython3.net/strings.html)
* [http://stackoverflow.com/questions/402283/stdwstring-vs-stdstring](http://stackoverflow.com/questions/402283/stdwstring-vs-stdstring)

The rest of this document is generally going to assume Unicode, unless stated otherwise.

## Unicode Characters in Strings
ROS 1 says string fields are to contain ASCII encoded data, but allows UTF-8.
DDS-XTYPES mandates UTF-8 be used as the encoding of IDL type `string`.
To be compatible with ROS 1 and DDS-XTYPES, in ROS 2 the content of a `string` is expected to be UTF-8.

## Introducing Wide Strings
ROS 2 messages will have a new [primitive field type](/articles/interface_definition.html) `wstring`.
This purpose is to allow ROS 2 nodes to comminicate with non-ROS DDS entities using an IDL containing a `wstring` field.
The encoding of data in this type should be UTF-16 to match DDS-XTYPES 1.2.
Since both UTF-8 and UTF-16 can encode the same code points, new ROS 2 messages should prefer `string` over `wstring`.

## Encodings are Recommendations
The choice of UTF-8 or UTF-16 for `string` and `wstring` are recommendations.
A `string` or `wstring` field is allowed to be populated and published with an unknown encoding.
If subscriber receives a string containing an unknown ncoding then it is up to the subscriber how to handle it.
It could decode it using an encoding that has been agreed to out of band.
Other subscribers like `ros2 topic echo` may echo the bytes in hexadecimal.

In practice using different encodings may not be possible.
The IDL specification forbids `string` from containing `NULL` values.
For compatibility a ROS message `string` field must not contain zero bytes, and a `wstring` field must not contain zero words.
An encoding like UTF-32 cannot be used since it may have zero bytes or words in a code point.
If UTF-32 encoded strings need to be transmitted then one should either convert to UTF-8 and use a `string` or use a `uint32` array.

<div class="alert alert-warning" markdown="1">
  <b>TODO:</b> Can ROS 1 publish a string with NULL bytes?
</div>

## Unicode Strings Across ROS 1 Bridge
Since ROS 1 and 2 both allow `string` to be UTF-8, the ROS 1 bridge will pass values unmodified between them.
If a message sent from ROS 2 to ROS 1 fails to serialize because a string contains values that are not legal UTF-8 then it won't be published by the bridge.

If a ROS 2 message has a field of type `wstring` then the bridge will attempt to convert it from UTF-16 to UTF-8.
The resulting UTF-8 encoded string will be published as a `string` type.
If the conversion fails then the bridge will not publish the message.

## Size of a Wide String

Both UTF-8 and UTF-16 are variable width encodings.
To minimize the amount of memory used, the `string` and `wstring` types are to be stored in client libraries according to the smallest possible code point.
This means `string` must be specified as a sequence of bytes, and `wstring` is to be specified as a sequence of words.

Some DDS implementations use 32bit types to store wide strings values.
This may be due to DDS-XTYPES 1.1 spec saying `wchar` is a 32bit value.
ROS 2 will instead aim to be compatible with DDS-XTYPES 1.2 and use 16bit storage for wide characters.
Generated code for ROS 2 messages will automatically handle the conversion when a message is serialized or deserialized.

### Bounded wide strings

Embedded systems accepting strings may use message definitions that restrict the maximum size of a string.
These are referred to as bounded strings.
Their purpose is to restrinct the amount of memory used, so the bounds must be specified as units of memory.
If a `string` field is bounded then the size is given in bytes.
Similarly the size of a bounded `wstring` is to be specified in words.
It is the responsibility of whoever populates a bounded `string` or `wstring` to make sure it contains whole code points only.
Partial code points are indistinuguishable from unknown encodings, so a bounded string whose last code point is incomplete will be published without error.

## Runtime impact of wide string

Dealing with wide strings puts more strain on the software of a system, both in terms of speed and code size.
UTF-8 and UTF-16 are both variable width encodings, meaning a code point can take 1 (UTF-8 only), 2, or 4 bytes to represent.
It may take multiple code points to represent a single user perceived character.
One of the goals of ROS 2 is to support microcontrollers that are contrained by both code size and processor speed.
Some wide string operations like splitting a string on a user perceived character may not be possible on these devices.

However, whole string equality checking is the same whether using wide strings or not.
Further splitting a UTF-8 string on an ASCII character is identical to splitting an ASCII character on an ASCII string.
If a microcontroller must do other types of string manipluation, then a `string` type should still be used.
Code in the subscriber callback should stop processing a message when it encounters a byte greater than 127 in a `string` field.

## What does the API look like to a user of ROS 2?

### Python 3

In Python 3 a string is a sequence of characters where each character can take 1 or more bytes.
Byte arrays are sequences of bytes.
Thus the ROS 2 API for dealing with `wstring` will take a string type in internally and convert it to UTF-16.

**Example**

```
import sys
from time import sleep

import rclpy

from rclpy.qos import qos_profile_default

from std_msgs.msg import WString

def main(args=None):
    if args is None:
        args = sys.argv

    rclpy.init(args)

    node = rclpy.create_node('talker')

    chatter_pub = node.create_publisher(WString, 'chatter', qos_profile_default)

    msg = String()

    i = 1
    while True:
        msg.data = 'Hello World: {0}'.format(i)
        i += 1
        print('Publishing: "{0}"'.format(msg.data))
        chatter_pub.publish(msg)
        # TODO(wjwwood): need to spin_some or spin_once with timeout
        sleep(1)


if __name__ == '__main__':
    main()
```

This code looks almost identical to the same demo for String.

### C++

In c++ wsing `wchar_t` has different sizes on different platforms (2 bytes on Windows, 4 bytes on Linux).
Instead ROS 2 will use `char16_t` for characters of wide strings, and `std::u16string` for wide strings themselves.

**Example**

```
#include <codecvt>
#include <iostream>
#include <memory>

#include "rclcpp/rclcpp.hpp"

#include "std_msgs/msg/wstring.hpp"

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);

  auto node = rclcpp::node::Node::make_shared("talker");

  rmw_qos_profile_t custom_qos_profile = rmw_qos_profile_default;
  custom_qos_profile.depth = 7;

  auto chatter_pub = node->create_publisher<std_msgs::msg::WString>("chatter", custom_qos_profile);

  rclcpp::WallRate loop_rate(2);

  auto msg = std::make_shared<std_msgs::msg::WString>();
  auto i = 1;
  std::wstring_convert<std::codecvt<char16_t, char, std::mbstate_t>, char16_t> convert_to_u16;

  while (rclcpp::ok()) {
    std::u16string hello(u"Hello World: " + convert_to_u16.from_bytes(std::to_string(i++)));
    msg->data = hello;
    std::cout << "Publishing: '" << msg->data.cpp_str() << "'" << std::endl;
    chatter_pub->publish(msg);
    rclcpp::spin_some(node);
    loop_rate.sleep();
  }

  return 0;
}
```