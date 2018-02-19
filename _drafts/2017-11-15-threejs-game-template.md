---
layout: post
title: "Introduction to the threejs-game template"
description: ""
comments: true
category: 
tags: [threejs, clojurescript, games]
---
{% include JB/setup %}

# Introduction


[three.js](https://github.com/mrdoob/three.js/) is a a Javascript 3D library written by [mrdoob](https://github.com/mrdoob). It is provides an abstraction over [WebGL](https://www.khronos.org/webgl/). This abstraction includes a scene to which objects are added. A render method displays a scene object using a camera object. Numerous methods, helpers and functions are included in the library to facilitate drawing objects in this scene. The render method is meant to be called each time the requestAnimationFrame method is called in order to display a 3D scene to the user. 

I've made a lein template for getting started using ClojureScript for game development. In order to keep things simple, it provides a starting point in the form of a simple 2D game where the goal of the game is to simply control the 'hero', a blue box,  with the keyboard arrow keys to get to the 'goal'.  

# The Game Loop

A fundamental component of any video game is the game loop. It is where the moments of the world tick by and actions occur. I've created an abstraction so that you do not need to think about setting up this time loop. The time loop is given an atom with a reference to a fn of Δt where

Δt = current-time - previous-time

current_time and previous-time refer to now and the last time requestAnimationFrame was called, respectively. Essentially, you will be creating functions where all the action occurs based upon some moment of time that has elapsed. 

# Initializing

Talk about initing the code

# Game Menus

Games do not typically throw you right into the action. They have menus which allow you to start, save and adjust settings on the game. The threejs-game template makes use of the React.js ClojureScript wrapper [reagent](http://reagent-project.github.io/) to create menus. 

# Starting the game

Talk about initializing the game

# Exploration of the time-fn

```clojure
{:objects
  {:Game {:description "A game owned by a developer for which scores can be recorded"
          :fields {:key {:type (non-null String)
                         :description "Unique identifier for this game"}
                   :name {:type (non-null String)}
                   :created {:type Int
                             :description "Unix epoch seconds when game was added to database"}}}}
 :queries
  {:game {:type :Game
          :description "Retrieve a single Game by its name"
          :args {:name {:type (non-null String)
                        :description "Unique name for game."}}
          :resolve :resolve-game}}}
```

