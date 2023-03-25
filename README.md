;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;; Innovation Ecosystem ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
extensions [table]

;;;;;;;;;;;;;;;;;;;;;; breeds ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

breed [entities entity]
breed [niches niche]

;;;;;;;;;;;;;;;;;;;;;; turtle variables ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

entities-own [

  ;; stores the scientific knowledge of the entity. It is a characteristic of Generators and Diffusers
  science-knowledge
  new-science-knowledge
  ;; lets the model know which entities have active scientific knowledge
  science?
  ;; stores the technological knowledge of the entity. It is a characteristic of Consumers and Diffusers
  tech-knowledge
  new-tech-knowledge
  ;; lets the  model know which entities have active technological knowledge
  technology?
  ;; stores the Hamming distance of the entity (currently just from one niche)
  sci-fitness
  tech-fitness
  fitness
  ;; stores the amount of resources kept by the entity
  resources
  ;; Stores the entity's reputation, given its resources and fitness in its niche
  reputation
  ;; does the entity assume a generator role in the ecosystem?
  generator?
  ;; does the entity assume a generator role in the ecosystem?
  consumer?
  ;; does the entity assume a generator role in the ecosystem?
  diffuser?
  ;; does the entity assume a generator role in the ecosystem?
  integrator?
  ;; entity's willingness to share knowledge with others
  willingness-to-share
  ;; entity's motivation to learn from others
  motivation-to-learn
  ;; entity's performance in creating oportunities to create new knowledge
  creation-performance
  ;; entity's performance in creating oportunities to develop science into technology
  development-performance
  ;; lets the model know if the agent performed crossover
  crossover?
  ;; lets the model know if the agent performed mutation
  mutation?
  ;; lets the model know if the agent performed development of science knowledge into technological knowledge
  development?
  ;; lets the model know if the agent shared knowledge as the emitter
  emitted?
  ;; lets the model know if the mutation attempt by a generator was successful
  mutated?
  ;; lets the model know if the integrator attempted to integrate
  integrated?
  ;; lets the model know if the interaction of an agent is ocurring through an integrator
  integration?
  ;; entities memory of past interactions with other agents
  interaction-memory

]

niches-own [

  ;; total-resources of a niche (put this on a slider in the future)
  niche-resources
  ;; stores the demand DNA of the niche
  niche-demand

]


;;;;;;;;;;;;;;;;;;;;;;; globals ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

globals [

  ;; holds counter value for which instruction is being displayed
  current-instruction
  ;; seed used to generate random-numbers
  my-seed
  ;; used to count the period of market stability between market mutations
  market-mutation-countdown

]

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; setup procedures ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;


to setup

  clear-all
  define-seed
  ask patches [set pcolor black];
  set-default-shape entities "circle";

  ;; creates the niches where entities will compete and assigns them a demand DNA
  ;; has to be created before the entities, so they can assess their fitness from the start
  create-market
  ;; creates the entities according to the inputs in the User Interface
  populate-ecosystem
  ;; resets the tick clock
  reset-ticks

end

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; go procedures   ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

to go

  ;; implements the stop trigger
  if ticks >= stop_trigger [
    stop
  ]

  ;; clears the links from previous iteration to keep a clean interface
  ask links [die]

  ;; asks entities to assess their Hamming distance for fitness test (check algoritm for Hamming)
  ask entities [test-fitness]

  ;; gives the entities resources proportional to their fitness, and collects resources
  ask entities [calculate-resource]

  ;; replaces dead entities with new startups, keeping the competition high
  ;; has to be called after calculate-resources, to avoid choosing parents that would die during the iteration
  if (count entities) !=  number_of_entities and startups? [
    spawn-startup (number_of_entities - (count entities))
  ]

  ;; stops the simulation if there is fewer than two entities with any kind of knowledge
  if count entities with [science? or technology?] < 2 [
    print "There are not enough knowledge entities left for interactions"
    stop
  ]

  ;; KNOWLEDGE ACTIVITIES

  ;; asks generators to perform research, in other words, mutate knowledge
  ;; must be called before the interact procedure to avoid loss of new knowledge
  ask entities with [generator?] [

    generate

  ]

  ;; asks entities with scientific and technological knowledge to develop science into technology
  ;; must be called before the interact procedure to avoid loss of new knowledge
  ask entities with [science? and technology?] [

    develop

  ]

  ;; KNOWLEDGE INTERACTION ACTIVITIES

  ;; asks integrators to facilitate the interaction and crossover between two entities
  ;; must be called before the interact procedure to avoid loss of new knowledge
  ask entities with [integrator?] [

    integrate

  ]

  ;; ask entities with some kind of knowledge  to look for similar partners and possibly, to crossover
  ;; it prevents entities who performed mutation, crossover or development to receive knowledge to protect
  ;; the results of these operations
  ask entities with [science? or technology?] [

    if resources > cost_of_crossover and not development? and not mutation? and not crossover? [

      interact

    ]
  ]

  ;; ask entities to update their knowledge given the actions performed during the iteration
  ask entities [

    set science-knowledge new-science-knowledge
    set tech-knowledge new-tech-knowledge

  ]

  ;; mutates the market if the countdown meets the number set in the interface
  market-mutation

  ;; ticks the iteration clock
  tick

end

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; external sources procedures ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;; it has to be on the same directory as the .nlogo model
;; _includes [ .nls]


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; entities' procedures ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
to populate-ecosystem

  ifelse random_ent_creation? [

    ;; creates random amounts of each kind of entity and assigns them resources, a knowledge DNA and others
    create-entities number_of_entities [

      ;; asks turtles to select their roles
      select-role
      set-entity-parameters
      set color blue

    ]
  ][
    ;; creates the selected amount of each kind of entity and assigns them resources, a knowledge DNA and others
    ;; creates pure generators
    create-entities number_of_generators [

      set generator? true
      set consumer? false
      set diffuser? false
      set integrator? false

      set-entity-parameters
      set color orange
    ]
    ;; creates pure consumers
    create-entities number_of_consumers [

      set generator? false
      set consumer? true
      set diffuser? false
      set integrator? false

      set-entity-parameters
      set color orange
    ]
    ;; creates pure diffusers
    create-entities number_of_diffusers [

      set generator? false
      set consumer? false
      set diffuser? true
      set integrator? false

      set-entity-parameters
      set color orange
    ]
    ;; creates pure integrators
    create-entities number_of_integrators [

      set generator? false
      set consumer? false
      set diffuser? false
      set integrator? true

      set-entity-parameters
      set color orange
    ]
    ;; creates consumers-generators
    create-entities number_of_cons_gen [

      set generator? true
      set consumer? true
      set diffuser? false
      set integrator? false

      set-entity-parameters
      set color orange
    ]
    ;; creates generators-diffusers
    create-entities number_of_gen_dif [

      set generator? true
      set consumer? false
      set diffuser? true
      set integrator? false

      set-entity-parameters
      set color orange

    ]
  ;; sets the total number of entities as the sum of the types created. It will allow the model to replace the numbers with randomly created startups
  ;; if startups? is set on at the interface
  set number_of_entities (number_of_generators + number_of_consumers + number_of_integrators + number_of_diffusers + number_of_cons_gen + number_of_gen_dif)

  ]

end



to evaluate-crossover [old-knowledge new-knowledge]

  let evaluation 0
  ;; the model currently has only one niche. If more than one niche is implemented, it will pick
  ;; one of the niches for the evaluation.

  ;; if there is an increase in fitness, the experience will be well evaluated (+ 0.10)
  ;; if there is no increase in fitness (remains the same or drops), it will be poorly evaluated (- 0.05)

  ;;;;;;;;;;;;;;; if the entities only evaluate for fitness;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
  ;; implements the function for all entities that are not consumers
  if evaluate_for_fitness? and not evaluate_for_learning? and not consumer? [
    let niche-demand-now [niche-demand] of one-of niches

  ;; compares the absolute fitness prior to the crossover, and after the crossover
    let fitness-old 0
    let fitness-new 0

    ;; assesses the complement of the hamming distance between the niche-demand and the knowledges
    ;; the higher the better
    set fitness-old knowledge - (hamming-distance old-knowledge niche-demand-now)
    set fitness-new knowledge - (hamming-distance new-knowledge niche-demand-now)

    ifelse fitness-new > fitness-old [
      ;; the case where there is an increase in fitness
      if motivation-to-learn < 1 [
        set evaluation 0.1
      ]
    ][
      ;; the case where there is no increase in fitness
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ]
  ]

  ;; implements the function for all entities that are consumers
  if evaluate_for_fitness_cons? and not evaluate_for_learning_cons? and consumer? [
    let niche-demand-now [niche-demand] of one-of niches

  ;; compares the absolute fitness prior to the crossover, and after the crossover
    let fitness-old 0
    let fitness-new 0

    ;; assesses the complement of the hamming distance between the niche-demand and the knowledges
    ;; the higher the better
    set fitness-old knowledge - (hamming-distance old-knowledge niche-demand-now)
    set fitness-new knowledge - (hamming-distance new-knowledge niche-demand-now)

    ifelse fitness-new > fitness-old [
      ;; the case where there is an increase in fitness
      if motivation-to-learn < 1 [
        set evaluation 0.1
      ]
    ][
      ;; the case where there is no increase in fitness
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ]
  ]


  ;;;;;;;;;;;;;;; if the entities only evaluate for learning;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
  ;; implements the function for all entities that are not consumers
  if evaluate_for_learning? and not evaluate_for_fitness? and not consumer? [
    ;; compares the absolute fitness prior to the crossover, and after the crossover
    ;; to assess if there was any learning
    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ][
      if motivation-to-learn < 1 [
        set evaluation 0.05
      ]
    ]
  ]

  ;; implements the function for all entities that are consumers
  if evaluate_for_learning_cons? and not evaluate_for_fitness_cons? and consumer? [
    ;; compares the absolute fitness prior to the crossover, and after the crossover
    ;; to assess if there was any learning
    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ][
      if motivation-to-learn < 1 [
        set evaluation 0.05
      ]
    ]
  ]

  ;;;;;;;;;;;;;;; if the entities evaluate for fitness and learning ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
  ;; implements the function for all entities that are not consumers
  if evaluate_for_fitness? and evaluate_for_learning? and not consumer? [

    ;; if there is no learning, the experience will be poorly evaluated (-0.05 in motivation)
    ;; if there is learning and there is an increase in fitness, it will be well evaluated (+ 0.05)
    ;; if there is learning but there is no increase in fitness, it will be indifferent (no changes in motivation)
    ;; if there is learning but there is decrease in fitness, it will be poorly evaluated (- 0.05)

    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      ;; the case with no learning
      if motivation-to-learn > 0 [
        set evaluation evaluation - 0.05
      ]
    ][
      ;; the case with learning
      if motivation-to-learn < 1 [
        let niche-demand-now [niche-demand] of one-of niches

        ;; compares the absolute fitness prior to the crossover, and after the crossover
        let fitness-old 0
        let fitness-new 0

        ;; assesses the complement of the hamming distance between the niche-demand and the knowledges
        ;; the higher the better
        set fitness-old knowledge - (hamming-distance old-knowledge niche-demand-now)
        set fitness-new knowledge - (hamming-distance new-knowledge niche-demand-now)

        ifelse fitness-new > fitness-old [
          ;; the case with learning and increase in fitness
          if motivation-to-learn < 1 [
            set evaluation evaluation + 0.1
          ]
        ][
          ;; the case of learning with no change or decrease in fitness
          if motivation-to-learn > 0 [
            set evaluation evaluation - 0.05
          ]
        ]
      ]
    ]
  ]

  ;; implements the function for all entities that are consumers
  if evaluate_for_fitness_cons? and evaluate_for_learning_cons? and consumer? [

    ;; if there is no learning, the experience will be poorly evaluated (-0.05 in motivation)
    ;; if there is learning and there is an increase in fitness, it will be well evaluated (+ 0.05)
    ;; if there is learning but there is no increase in fitness, it will be indifferent (no changes in motivation)
    ;; if there is learning but there is decrease in fitness, it will be poorly evaluated (- 0.05)

    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      ;; the case with no learning
      if motivation-to-learn > 0 [
        set evaluation evaluation - 0.05
      ]
    ][
      ;; the case with learning
      if motivation-to-learn < 1 [
        let niche-demand-now [niche-demand] of one-of niches

        ;; compares the absolute fitness prior to the crossover, and after the crossover
        let fitness-old 0
        let fitness-new 0

        ;; assesses the complement of the hamming distance between the niche-demand and the knowledges
        ;; the higher the better
        set fitness-old knowledge - (hamming-distance old-knowledge niche-demand-now)
        set fitness-new knowledge - (hamming-distance new-knowledge niche-demand-now)

        ifelse fitness-new > fitness-old [
          ;; the case with learning and increase in fitness
          if motivation-to-learn < 1 [
            set evaluation evaluation + 0.1
          ]
        ][
          ;; the case of learning with no change or decrease in fitness
          if motivation-to-learn > 0 [
            set evaluation evaluation - 0.05
          ]
        ]
      ]
    ]
  ]


  ;; incorporates the evaluation into the motivation-to-learn
  set motivation-to-learn motivation-to-learn + evaluation

  ;; limits motivation-to-learn within the bounds of 0 an 1
  ifelse motivation-to-learn > 1 [
    set motivation-to-learn 1
  ][
    if motivation-to-learn < 0 [
      set motivation-to-learn 0
    ]
  ]


end

to evaluate-crossover-dual [older-tech-knowledge newer-tech-knowledge older-science-knowledge newer-science-knowledge]

  let evaluation 0
  ;; the model currently has only one niche. If more than one niche is implemented, it will pick
  ;; one of the niches for the evaluation.

  ;; if there is an increase in fitness, the experience will be well evaluated (+ 0.10)
  ;; if there is no increase in fitness (remains the same or drops), it will be poorly evaluated (- 0.05)

  ;;;;;;;;;;;;;;; if the entities only evaluate for fitness;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
  ;; implements the function for all entities that are not consumers
  if evaluate_for_fitness? and not evaluate_for_learning? and not consumer? [
    let niche-demand-now [niche-demand] of one-of niches

    ;; compares the absolute fitness prior to the crossover, and after the crossover
    let fitness-old 0
    let fitness-new 0

    ;; assesses the complement of the hamming distance between the niche-demand and the knowledges
    ;; the higher the better
    ;; in order to do so a single list is created containing both tech and science DNAs, and that is
    ;; compared to a doubled niche-demand-now
    let new-knowledge 0
    let old-knowledge 0

    set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
    set old-knowledge sentence older-science-knowledge older-tech-knowledge
    let double-demand sentence niche-demand-now niche-demand-now

    set fitness-old knowledge - (hamming-distance old-knowledge double-demand)
    set fitness-new knowledge - (hamming-distance new-knowledge double-demand)


    ifelse fitness-new > fitness-old [
      ;; the case where there is an increase in fitness
      if motivation-to-learn < 1 [
        ;; there is only one possible positive outcome
        set evaluation 0.1
      ]
    ][
      ;; the case where there is no increase in fitness - it either stays the same or decreases
      ;; both outcomes are negative
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ]
  ]

  ;; implements the function for all entities that are consumers
  if evaluate_for_fitness_cons? and not evaluate_for_learning_cons? and consumer? [
    let niche-demand-now [niche-demand] of one-of niches

    ;; compares the absolute fitness prior to the crossover, and after the crossover
    let fitness-old 0
    let fitness-new 0

    ;; assesses the complement of the hamming distance between the niche-demand and the knowledges
    ;; the higher the better
    ;; in order to do so a single list is created containing both tech and science DNAs, and that is
    ;; compared to a doubled niche-demand-now
    let new-knowledge 0
    let old-knowledge 0

    set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
    set old-knowledge sentence older-science-knowledge older-tech-knowledge
    let double-demand sentence niche-demand-now niche-demand-now

    set fitness-old knowledge - (hamming-distance old-knowledge double-demand)
    set fitness-new knowledge - (hamming-distance new-knowledge double-demand)


    ifelse fitness-new > fitness-old [
      ;; the case where there is an increase in fitness
      if motivation-to-learn < 1 [
        ;; there is only one possible positive outcome
        set evaluation 0.1
      ]
    ][
      ;; the case where there is no increase in fitness - it either stays the same or decreases
      ;; both outcomes are negative
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ]
  ]



  ;;;;;;;;;;;;;;; if the entities only evaluate for learning ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
  ;; implements the function for all entities that are not consumers
  if evaluate_for_learning? and not evaluate_for_fitness? and not consumer? [
    ;; compares the entities' new tech and science DNA bit by bit with its previous version
    ;; to assess if there was any learning
    ;; both knowledges are tested
    let new-knowledge 0
    let old-knowledge 0

    set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
    set old-knowledge sentence older-science-knowledge older-tech-knowledge

    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ][
      if motivation-to-learn < 1 [
        set evaluation 0.05
      ]
    ]
  ]

  ;; implements the function for all entities that are consumers
  if evaluate_for_learning_cons? and not evaluate_for_fitness_cons? and consumer? [
    ;; compares the entities' new tech and science DNA bit by bit with its previous version
    ;; to assess if there was any learning
    ;; both knowledges are tested
    let new-knowledge 0
    let old-knowledge 0

    set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
    set old-knowledge sentence older-science-knowledge older-tech-knowledge

    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      if motivation-to-learn > 0 [
        set evaluation -0.05
      ]
    ][
      if motivation-to-learn < 1 [
        set evaluation 0.05
      ]
    ]
  ]

  ;;;;;;;;;;;;;;; if the entities evaluate for fitness and learning  ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
  ;; implements the function for all entities that are not consumers
  if evaluate_for_fitness? and evaluate_for_learning? and not consumer? [

    ;; if there is no learning, the experience will be poorly evaluated (-0.05 in motivation)
    ;; if there is learning and there is an increase in fitness, it will be well evaluated (+ 0.1)
    ;; if there is learning but there is no increase in fitness, it will be poorly evaluated (- 0.05)
    ;; if there is learning but there is decrease in fitness, it will be poorly evaluated (- 0.05)
    let new-knowledge 0
    let old-knowledge 0

    set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
    set old-knowledge sentence older-science-knowledge older-tech-knowledge

    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      ;; the case with no learning
      if motivation-to-learn > 0 [
        set evaluation evaluation - 0.05
      ]
    ][
      ;; the case with learning
      let niche-demand-now [niche-demand] of one-of niches
      ;; compares the absolute fitness prior to the crossover, and after the crossover
      let fitness-old 0
      let fitness-new 0
      let double-demand sentence niche-demand-now niche-demand-now

      set fitness-old knowledge - (hamming-distance old-knowledge double-demand)
      set fitness-new knowledge - (hamming-distance new-knowledge double-demand)

      ifelse fitness-new > fitness-old [
        ;; the case with learning and increase in fitness
        if motivation-to-learn < 1 [
          set evaluation evaluation + 0.1
        ]
      ][
        ;; the case of learning with no change or decrease in fitness
        if motivation-to-learn > 0 [
          set evaluation evaluation - 0.05
        ]
      ]
    ]
  ]

  ;; implements the function for all entities that are consumers
  if evaluate_for_fitness_cons? and evaluate_for_learning_cons? and consumer? [

    ;; if there is no learning, the experience will be poorly evaluated (-0.05 in motivation)
    ;; if there is learning and there is an increase in fitness, it will be well evaluated (+ 0.1)
    ;; if there is learning but there is no increase in fitness, it will be poorly evaluated (- 0.05)
    ;; if there is learning but there is decrease in fitness, it will be poorly evaluated (- 0.05)
    let new-knowledge 0
    let old-knowledge 0

    set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
    set old-knowledge sentence older-science-knowledge older-tech-knowledge

    ifelse (hamming-distance old-knowledge new-knowledge) = 0 [
      ;; the case with no learning
      if motivation-to-learn > 0 [
        set evaluation evaluation - 0.05
      ]
    ][
      ;; the case with learning
      let niche-demand-now [niche-demand] of one-of niches
      ;; compares the absolute fitness prior to the crossover, and after the crossover
      let fitness-old 0
      let fitness-new 0
      let double-demand sentence niche-demand-now niche-demand-now

      set fitness-old knowledge - (hamming-distance old-knowledge double-demand)
      set fitness-new knowledge - (hamming-distance new-knowledge double-demand)

      ifelse fitness-new > fitness-old [
        ;; the case with learning and increase in fitness
        if motivation-to-learn < 1 [
          set evaluation evaluation + 0.1
        ]
      ][
        ;; the case of learning with no change or decrease in fitness
        if motivation-to-learn > 0 [
          set evaluation evaluation - 0.05
        ]
      ]
    ]
  ]

  ;; incorporates the evaluation into the motivation-to-learn
  set motivation-to-learn motivation-to-learn + evaluation

  ;; limits motivation-to-learn within the bounds of 0 an 1
  ifelse motivation-to-learn > 1 [
    set motivation-to-learn 1
  ][
    if motivation-to-learn < 0 [
      set motivation-to-learn 0
    ]
  ]


end

;; creates startups (all pure consumers or generators-consumers)
to spawn-startup [number-of-startups]

  repeat number-of-startups[
    create-entities 1 [

      set generator? false
      if consumer_generator_startups? [
        set generator? one-of [true false]
      ]
      ;; set generator? one-of [true false] ;; if a chance of creating a generator consumer is desired
      set consumer? true
      set diffuser? false
      set integrator? false

      ;; assigns all other variables, as well as a random tech-knowledge and science-knowledge DNA
      set-entity-parameters

      ;; chooses one of the other entities to be a parent of the new startup
      let parent1 choose-partner

      ;; chooses the second parent with replacement
      let parent2 choose-partner

      ;; the knowledge code will work only if there are two suitable parents available
      if parent1 != nobody and parent2 != nobody [

        ;; if the startup and both parents have scientific knowledge
        ;; if the startup has scientific knowledge and none of the parents has scientific
        ;; knowledge, the scientific knowledge randomly set by set-entity-parameters will be kept
        ifelse science? and [science?] of parent1 and [science?] of parent2 [

          ;; bits1 is the science-knowledge of the parent 1
          let bits1 [science-knowledge] of parent1
          ;; bits2 is the science-knowledge of the parent 2
          let bits2 [science-knowledge] of parent2
          set new-science-knowledge crossover bits1 bits2
          ;; also performs a mutation in science knowledge
          set new-science-knowledge mutate new-science-knowledge

        ][
          ;; if the startup and only parent 1 have scientific knowledge
          ifelse science? and [science?] of parent1 [

            ;; the knowledge of the suitable parent is copied
            set new-science-knowledge [science-knowledge] of parent1
            ;; also performs a mutation in science knowledge
            set new-science-knowledge mutate new-science-knowledge

          ][
            ;; if the startup and only parent 2 have scientific knowledge
            if science? and [science?] of parent2 [

              ;; the knowledge of the suitable parent is copied
              set new-science-knowledge [science-knowledge] of parent2
              ;; also performs a mutation in science knowledge
              set new-science-knowledge mutate new-science-knowledge

            ]
          ]
        ]

        ;; deals with the technologic knowledge of startups
        ifelse technology? and [technology?] of parent1 and [technology?] of parent2 [

          ;; bits1 is the tech-knowledge of the parent 1
          let bits1 [tech-knowledge] of parent1
          ;; bits2 is the tech-knowledge of the emitter
          let bits2 [tech-knowledge] of parent2
          set new-tech-knowledge crossover bits1 bits2

          ;; also performs a mutation in technological knowledge
          set new-tech-knowledge mutate new-tech-knowledge

        ][
          ;; if the startup and only parent 1 have scientific knowledge
          ifelse technology? and [technology?] of parent1 [

            ;; the knowledge of the suitable parent is copied
            set new-tech-knowledge [tech-knowledge] of parent1
            ;; also performs a mutation in science knowledge
            set new-tech-knowledge mutate new-tech-knowledge

          ][
            ;; if the startup and only parent 2 have scientific knowledge
            if technology? and [technology?] of parent2 [

              ;; the knowledge of the suitable parent is copied
              set new-tech-knowledge [tech-knowledge] of parent2
              ;; also performs a mutation in science knowledge
              set new-tech-knowledge mutate new-tech-knowledge

            ]
          ]
        ]

        ;; picks the knowledge activities parameters of one of the parents
        ifelse random-float 1 < 0.5 [
          set willingness-to-share [willingness-to-share] of parent1
        ][
          set willingness-to-share [willingness-to-share] of parent2
        ]

        ifelse random-float 1 < 0.5 [
          set motivation-to-learn [motivation-to-learn] of parent1
        ][
          set motivation-to-learn [motivation-to-learn] of parent2
        ]

        ifelse random-float 1 < 0.5 [
          set creation-performance [creation-performance] of parent1
        ][
          set creation-performance [creation-performance] of parent2
        ]

        ifelse random-float 1 < 0.5 [
          set development-performance [development-performance] of parent1
        ][
          set development-performance [development-performance] of parent2
        ]
      ]

      ;; finishes by making both new-knowledge and knowledge variables equal, as the entity is starting its life and has not yet learned
      set science-knowledge new-science-knowledge
      set tech-knowledge new-tech-knowledge

      test-fitness
      set color cyan

    ]
  ]

end

;; transforms scientific knowledge into technological knowledge
to develop

  if resources > cost_of_development [
      if random-float 1 < development-performance [

        ;; using new-tech-knowledge instead of tech-knowledge allows several knowledge activities to be performed without loosing the notion of paralelism
        ;; although some of the learning of the previous activity may be altered

        set new-tech-knowledge crossover new-tech-knowledge new-science-knowledge


        ;; flags the model that internal crossover between scientific and technologica knowledge (development) was attempted
        set development? true
      ]
    ]

end

;; creates new knowledge through mutation
to generate

  if resources > cost_of_mutation [
      if random-float 1 < creation-performance [
        set mutation? true
        let new-science-knowledge-mut new-science-knowledge
        set new-science-knowledge mutate new-science-knowledge
        if length ( remove true ( map [ [a b] -> a = b ] new-science-knowledge-mut new-science-knowledge )  ) > 0 [
          set mutated? true
        ]
      ]
    ]
end

to set-entity-parameters

  ;; sets knowledge as false by default, to be changed according to the roles
  set science? false
  set technology? false

  ;; sets the type of knowledge the entity has according to its role

  if generator? [set science? true]
  if consumer? [set technology? true]
  if diffuser? [
    ;; flips the coin untill the diffuser has some kind of knowledge, science, tech or both
    ;;while [ not science? and not technology?] [
      ;;if not science? [set science? one-of [true false]]
      ;;if not technology? [set technology? one-of [true false]]
    ;;]
    set science? true
    set technology? true

  ]

  ;; gives the entities its initial resources
  set resources initial_resources
  set size (initial_resources / 500)
  setxy random-xcor random-ycor

  ;; sets the individual characteristics of the entities that will influence how often they interact with others
  ;; this is done as a normal distribution
  set willingness-to-share random-normal willingness_to_share std_dev_willingness
  set motivation-to-learn random-normal motivation_to_learn std_dev_motivation
  set creation-performance random-normal creation_performance std_dev_creation_performance
  set development-performance random-normal development_performance std_dev_development_performance

  ;; tells the model they the entities have not performed any of these actions yet
  ;; Flags that impact on the payment of resources and measurement of activities
  set crossover? false
  set mutation? false
  set integration? false
  set development? false

  ;; Flags that impact on the receiving of resources and measurement of activities
  set emitted? false
  set mutated? false
  set integrated? false

  ;; creates a table to implement the interaction memory
  set interaction-memory table:make

  ;; selects the shape of the entity given its role in the ecosystem
  select-shape
  create-knowledge-DNA
  test-fitness

end

;; creates a superfit entity, perhaps an entity that comes from another market
to create-super-generator


  create-entities 1 [

    set generator? true
    set consumer? false
    set diffuser? false
    set integrator? false

    set-entity-parameters

    ;; sets the creation performance to 0 and the motivation to learn to 0 to preserve the super-fitness
    if not super_share? [
      set creation-performance 0
      set motivation-to-learn 0
    ]

    ;; creates a perfect match to the market demand
    set science-knowledge [niche-demand] of one-of niches
    set new-science-knowledge science-knowledge

    ;; assigns the supercompetitor the best fitness score possible from the start
    test-fitness
    set color magenta
    set shape "star 2"

  ]

end

to create-super-competitor


  create-entities 1 [

    set generator? false
    set consumer? true
    set diffuser? false
    set integrator? false

    set-entity-parameters

    ;; sets the motivation to learn and willingness to share to 0 to preserve the competitivity of the super competitor
    if not super_share? [
      set willingness-to-share 0
      set motivation-to-learn 0
    ]

    ;; creates a perfect match to the market demand
    set tech-knowledge [niche-demand] of one-of niches
    set new-tech-knowledge tech-knowledge

    ;; assigns the supercompetitor the best fitness score possible from the start
    test-fitness
    set color magenta
    set shape "square 2"
  ]

end


;; creates a superfit diffuser. Different from the other super entities, this one assumes
to create-super-diffuser


  create-entities 1 [

    set generator? false
    set consumer? false
    set diffuser? true
    set integrator? false

    set-entity-parameters

    ;; a super diffuser gets maximum efficiency when sharing knowledge
    if not super_share? [
      set willingness-to-share 1
      set motivation-to-learn 0
    ]

    ;; creates a perfect match to the market demand
    set tech-knowledge [niche-demand] of one-of niches
    set new-tech-knowledge tech-knowledge
    set science-knowledge tech-knowledge
    set new-science-knowledge science-knowledge

    ;; assigns the supercompetitor the best fitness score possible from the start
    test-fitness
    set color magenta
    set shape "triangle 2"
  ]

end


to select-role

  ;; randomly sets the role (s) an entity assumes in the ecosystem.
  ;; sets the type of knowledge the entity has according to its role
  ;; later it has to be more controllable, assigning a known proportion of each

  ;; does the entity assume a GENERATOR role in the ecosystem?
  set generator? one-of [true false]

  ;; does the entity assume a CONSUMER role in the ecosystem?
  set consumer? one-of [true false]

  ;; does the entity assume a DIFFUSER role in the ecosystem?
  ;; if the entity accumulates other role, it will retain the knowledge the other role confers, and maybe add another
  set diffuser? one-of [true false]

  ;; does the entity assume an INTEGRATOR role in the ecosystem? Integrators don't need to have scientific or technological knowledge
  set integrator? one-of [true false]

  ;; The code above randomly assigns roles, and they may be cumulative.
  ;; If any entity remains without a role in the ecosystem, it will be turned into a CONSUMER
  if not generator? and not consumer? and not diffuser?  and not integrator? [ set consumer? true ]

end

;; flips a biased coin with the given probability of showing 1
to-report flip-of-a-coin [probability]
  ifelse random-float 1 < probability [
    report 1
  ][
    report 0
  ]
end

to create-knowledge-DNA
  ;; randomly creates the scientific knowledge string
  ;; if the entity does not possess this kind of knowledge, the string is all 0's
  ;; it also initializes the new-science-knowledge
  ;; if the ordered_DNA? option is selected, it sorts the entities DNA, leaving a blank area in the DNA for
  ;; knowledge not yet learned/existing in the ecossistem
  ;; although very similar, the entities will still have slight differences between each other

  ifelse science? [
    ;; set science-knowledge n-values knowledge [random 2]
    set science-knowledge n-values knowledge  [ flip-of-a-coin initial_fitness_probability ]
    if ordered_DNA? [
      set science-knowledge sort science-knowledge
    ]
    set new-science-knowledge science-knowledge

  ][
    set science-knowledge n-values knowledge [0]
    set new-science-knowledge science-knowledge
  ]

  ;; randomly creates the technological knowledge string
  ;; if the entity does not possess this kind of knowledge, the string is all 0's
  ;; it also initializes the new-tech-knowledge
  ;; if the ordered_DNA? option is selected, it sorts the entities DNA, leaving a blank area in the DNA for
  ;; knowledge not yet learned/existing in the ecossistem
  ;; although very similar, the entities will still have slight differences between each other

  ifelse technology? [
    ;; set tech-knowledge n-values knowledge [random 2]
    set tech-knowledge n-values knowledge [ flip-of-a-coin initial_fitness_probability ]
    if ordered_DNA? [
      set tech-knowledge sort tech-knowledge
    ]
    set new-tech-knowledge tech-knowledge
  ][
    set tech-knowledge n-values knowledge [0]
    set new-tech-knowledge tech-knowledge
  ]

end

;; reports the hamming distance between two strings
to-report hamming-distance [bits1 bits2]
  let h-distance 0
  set h-distance length remove true (map [ [ a b ] -> a = b ] bits1 bits2 )
  report h-distance
end


;; evaluates the complement of the hamming distance between the niche's demand, the tech-knowledge and the sci-knowledge
to test-fitness

  set fitness 0
  set sci-fitness 0
  set tech-fitness 0
  let niche-demand-now [niche-demand] of one-of niches

  set tech-fitness knowledge - (hamming-distance tech-knowledge niche-demand-now)
  set sci-fitness knowledge - (hamming-distance science-knowledge niche-demand-now )
  set fitness max (list tech-fitness sci-fitness)

  ;; sets the color of the entities based on its absolute fitness
  select-fitness-color

end

;; procedure to calculate how much must the entity receive from the market, and how much must it pay to live
;; also adjusts the size of the entity given the amount of its resources
;; *** create options to award resources to each role

to calculate-resource

  ;; Awards the entities resources based on their actions / fitness

  ;; gives CONSUMER entities a share of the niche's resources proportional to its market share (relative tech-fitness)
  ;; the relative fitness is calculated of the tech-fitness of entities who compete for market share (CONSUMERS of knowledge)
  if consumer? [
    set resources resources + (niche_resources * (tech-fitness / (sum [tech-fitness] of entities with [consumer?])))
  ]

  ;;*** equation that allows consumers to compete against a standard, and not against each other. Meet the minimum and you are alive.
  ;;if consumer? [
  ;;  set resources resources + (niche_resources * tech-fitness / knowledge)
  ;;]

  ;; pays emitters for their knowledge
  if emitted? [
      set resources resources + (cost_of_crossover / 2)
      set emitted? false
  ]

  ;;******************* new function for resources of non market entities

  ;; Gives non market entities the minimum resources to live, to keep them always alive
  ;; The entities will receive extra resources if they suceed in sharing resources, generating new knowledge
  if not consumer? [

   ;; if the mutation is well suceeded, the generator has the budget renewed.
   ;; admits that a research facility receives, besides the cost of research, operational and capital funds.
   ;; consumers mutate to increase their own competitivity
    if mutated? [
      set resources resources + (10 * cost_of_mutation)
      set mutated? false
    ]
  ]

  ;; takes resources from the entity proportionally to its total amount of resources, respecting the minimum amount to stay alive
  ;; the amount necessary grows with the amount of resources the entity amasses (which is the growth of the entity)
  ;; the rate of the expense growth is given by the expense to live growth slider
  ;; caveat - this keeps the non_economical at a minimum resource status, which may hamper their chances to be selected as partners unless
  ;; unless they are really fit.

  ifelse not non_economical_entities? [
    set resources resources - (minimum_resources_to_live + (resources * expense_to_live_growth))
  ][
    if consumer? [
      set resources resources - (minimum_resources_to_live + (resources * expense_to_live_growth))
    ]
  ]

  ;; Collects resources for the attempts of action
  ;; if the entity attempted to crossover, collect its cost
  if crossover? [
    set resources resources - cost_of_crossover
    set crossover? false
  ]

  ;; if the entity attempted to mutate, collect its cost
  if mutation? [
    set resources resources - cost_of_mutation
    set mutation? false
  ]

  ;; if the entity attempted to convert scientific knowledge into technological knowledge, collect its cost
  if development? [
    set resources resources - cost_of_development
    set development? false
  ]

  ;; Resets the integration attempt counter
  set integrated? false
  set integration? false

  ;; sets the size of the entity given its accumulated amount of resources
  set-size-entity

  ;; kills the entity if it has no resources left
  if resources < 0 [
    die
  ]

end


;; creates agentsets of possible partners who possess the same kind of knowledge possessed by the choosing entity
;; *** issue - something has to be done with the agentsets before it is closed. The lottery for an instance

to-report choose-partner

  let possible-partners nobody
  ;; creates an agentset with entities possessing knowledge similar to the knowledge of the choosing entity
  ifelse science? and technology? [
    set possible-partners other entities with [science? or technology?]
    ][
      ifelse science? [
      set possible-partners other entities with [science?]
      ][
        if technology? [
        set possible-partners other entities with [technology?]
        ]
      ]
    ]

  ;; creates roulette that will select the partner from the agentset of suitable partners (Lottery Example model from Netlogo)
  ;; the method favours those with higher reputation and more resouces, but it doesnt rule anyone out.

  ;; sums the fitness and resources of all possible partners to perform the normalization
  let total-fitness sum [fitness] of possible-partners
  let total-resources sum [resources] of possible-partners

  ;; this represents the sum of 100% of the normalized reputation and 100% of the normalized resources
  ;; but with less computational cost
  let pick random-float (1 + 1)
  let partner nobody
  ask possible-partners [
    ;; if there's no winner yet...
    if partner = nobody [
      ;; gives the chance of the entity given the sum of its normalized resources and normalized fitness
      ;; if there is a memory of having interacted with that entity in the past, it also boosts the chances of the agent
      ;; to be selected. The myself command uses the interaction-memory of the entity who is calling the choose-partner procedure
      ifelse table:has-key? [interaction-memory] of myself who [

        ifelse ((resources / total-resources) + (fitness / total-fitness) + (table:get [interaction-memory] of myself who)) > pick [
          set partner self
        ]
        [
          set pick pick - ((resources / total-resources) + (fitness / total-fitness))
        ]
      ][
        ifelse ((resources / total-resources) + (fitness / total-fitness)) > pick [
          set partner self
        ]
        [
          set pick pick - ((resources / total-resources) + (fitness / total-fitness))
        ]
      ]
    ]
  ]


  report partner

  ;; *** alternate code for simplicity
  ;; lottery example commands	rnd:weighted-one-of	rnd:weighted-one-of-list
  ;; The idea behind this procedure is a bit tricky to understand.
  ;; Basically we take the sum of the sizes of the turtles and the sum of their best fitness between science and technology,
  ;; and that's how many "tickets" we have in our lottery.  Then we pick
  ;; a random "ticket" (a random number).  Then we step through the shorter code option - see netlogo online manual
  ;; ask rnd:weighted-one-of entities with science? [ resources + fitness ] [set partner self]

end

 ;; This procedure implements the attempt to interact with other entities
 ;; it will call procedures so select suitable partners, and from this pool, to select one
 ;; will analyze what kind of knowledge can be used for crossover and call the operation
 ;; it will then store the result in the new-knowledge variable, which will be used to update the
 ;; entity's knowledge at the end of the iteration.
 ;; it cannot update it immediatly because it would tamper with the fitness evaluation performed by other entities
 ;; before the run is done, giving the impression of instantaneous learning.

to interact

  ;; given the receiver's motivation to learn
  ;; chooses a suitable partner to be the emitter
  ;; If the interaction is intermediated by an integrator, there is a receiver's motivation-to-learn boost, increasing the chance of interaction

  let motivation-to-learn-actual 0
  let willingness-to-share-actual 0
  let receiver self

  ;; if this interaction is happening through an integrator, boos the motivation to learn
  ifelse integration? [
    ;; uses the integration_boost from the slider in the interface
    set motivation-to-learn-actual (motivation-to-learn + integration_boost)
  ]
  [
    set motivation-to-learn-actual motivation-to-learn
  ]

  ;; if the receiver decides, given its motivation (boosted or not) to interact and it has resources, look for partner
  ifelse (random-float 1 < motivation-to-learn-actual) and (resources > cost_of_crossover) [
    let partner choose-partner

    ;; if a emitter partner is found and the interaction is happening through an integrator, boost its willingness to share
    ;; it also checks if the partner is fit enough to be accepted
if partner != nobody and not ( [fitness] of partner < fitness) [
      ifelse integration? [
        set willingness-to-share-actual ([willingness-to-share] of partner + integration_boost)
      ][
        set willingness-to-share-actual [willingness-to-share] of partner
      ]

      ;; adds to the willingness to share of the chosen partner the interaction memory the partner has of the receiver
      if (table:has-key? [interaction-memory] of partner [who] of receiver) [
        set willingness-to-share-actual (willingness-to-share-actual + table:get [interaction-memory] of partner [who] of receiver )
      ]
    ]

    ;; given the partners willingness to share (boosted or not), begin crossover
    ifelse partner != nobody and (random-float 1 < willingness-to-share-actual) [
      ;;  asks the partner to create a directional link to the receiver
      ask partner [
        create-link-to receiver
        set emitted? true
      ]

      set crossover? true

      ;; *** decide whether an interaction between entities with both kinds of knowledge results in changes in both
      ;; kinds of knowledge
      ;; if both the entity (receiver) and the partner (emitter) possess scientific and technological knowledge
      ifelse science? and technology? and [science? and technology?] of partner [

        ;; bits1 is the science-knowledge of the receiver
        let bits1 science-knowledge
        ;; bits2 is the science-knowledge of the emitter
        let bits2 [science-knowledge] of partner
        set new-science-knowledge crossover bits1 bits2

        ;; after learning has been done, also performs a mutation in science knowledge, following traditional genetic algorithms
        set new-science-knowledge mutate new-science-knowledge

        ;; bits1 is the tech-knowledge of the receiver
        set bits1 tech-knowledge
        ;; bits2 is the tech-knowledge of the emitter
        set bits2 [tech-knowledge] of partner
        set new-tech-knowledge crossover bits1 bits2

        ;;*** repensar os efeitos deste trecho do código. E se não houver transf de tech, mas sim de sci?
        ;; existem 4 possibilidades para a imagem deste link: os dois aprenderam, apenas sci aprendeu, apenas tech aprendeu, ninguém aprendeu
        ;; pode-se usar também 4 tipos de links, um sólido, um pontilhado, um tracejado e um vermelho
        ;; talvez eu tenha que pensar num update-link appearance para o caso dual, assim como o evaluate crossover.
        update-link-appearance-dual tech-knowledge new-tech-knowledge science-knowledge new-science-knowledge yellow

        evaluate-crossover-dual tech-knowledge new-tech-knowledge science-knowledge new-science-knowledge


      ][;; if both the entity (receiver) and the partner (emitter) possess only scientific knowledge

        ifelse science? and [science?] of partner [
          ;; bits1 is the science-knowledge of the receiver
          let bits1 science-knowledge
          ;; bits2 is the science-knowledge of the emitter
          let bits2 [science-knowledge] of partner
          set new-science-knowledge crossover bits1 bits2

          ;; after learning has been done, also performs a mutation in science knowledge, following traditional genetic algorithms
          set new-science-knowledge mutate new-science-knowledge
          update-link-appearance new-science-knowledge science-knowledge green

          evaluate-crossover science-knowledge new-science-knowledge

        ][;; if both the entity (receiver) and the partner (emitter) possess only technological knowledge
          ;; the code ignores those who don't have any knowledge, but these have been ignored already by the choose-partner procedure

          if technology? and [technology?] of partner [
            ;; bits1 is the tech-knowledge of the receiver
            let bits1 tech-knowledge
            ;; bits2 is the tech-knowledge of the emitter
            let bits2 [tech-knowledge] of partner
            set new-tech-knowledge crossover bits1 bits2
            update-link-appearance new-tech-knowledge tech-knowledge blue

            evaluate-crossover tech-knowledge new-tech-knowledge

          ]
        ]
      ]

      ;; inserts a memory of this interaction in the receiver's memory
      ;; the value is currently given by the parameter on the interface  trust_in_known_partners
      ;; but it can also be done with the result of the evaluation of the crossover or other criteria
      table:put interaction-memory [who] of partner trust_in_known_partners
      ;; inserts a memory of this interaction in the emitter's (partner) memory
      table:put [interaction-memory] of partner who trust_in_known_partners


    ][;; the crossover failed the test of the willingness-to-share-actual or the search for a partner
      ;; in either case the integration, if occurred, failed
      set integration? false
    ]
  ][;; the crossover failed the test of the motivation-to-learn-actual or there are not enough resources
    ;; in either case the integration, if occurred, failed
    set integration? false
  ]

end

;; The integrator facilitates interaction
;; The integrator finds an entity asks it to find a partner.
;; It then boosts the willingness to share an motivation to learn of both of them, facilitating the transaction
to integrate

  let partner1 one-of other entities with [science? or technology?]
  if partner1 != nobody and not crossover? [
    ask partner1 [
      ;; Set integration? on the integrated knowledge entity, signaling it has been approached
      ;; by an integrator
      set integration? true
      interact
    ]

    ;; Set integrated? in the integrator, signalling it attempted to integrate entities
    set integrated? true

  ]

end

;; Crossover procedure from simple genetic algorithm model
;; This reporter performs one-point crossover on two lists of bits.
;; That is, it chooses a random location for a splitting point.
;; Then it reports two new lists, using that splitting point,
;; by combining the first part of bits1 with the second part of bits2
;; and the first part of bits2 with the second part of bits1;
;; it puts together the first part of one list with the second part of
;; the other.
;; In this model, if we consider unidirectional exchanges of knowledge, only
;; one of the answers has to be chosen to represent the new knowledge DNA of
;; the receiver entity
;; reports one of two strings of bits resulting from single point crossover

to-report crossover [bits1 bits2]

  let split-point 1 + random (length bits1 - 1)
  report item one-of [0 1]
    list (sentence (sublist bits1 0 split-point)
                   (sublist bits2 split-point length bits2))
         (sentence (sublist bits2 0 split-point)
                   (sublist bits1 split-point length bits1))

end

;; mutation procedure from simple genetic algorithm model
;; This procedure causes random mutations to occur in a solution's bits.
;; The probability that each bit will be flipped is controlled by the
;; MUTATION_RATE slider.
;; The cost to mutate is not charged here because mutation may be a by product of learning through crossover
;; or the result of efforts in research. The costs of the first are included in the crossover costs
;; the costs of the second are charged when the mutate procedure is called in the go function
;; the function is also used in the creation of startups

to-report mutate [bits]

   report map [ [b] -> ifelse-value (random-float 100.0 < mutation_rate) [
       1 - b
     ]
     [
       b
     ]
   ] bits

end

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; niche's procedures ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
to create-market
  set market-mutation-countdown 0

  create-niches 1 [

    ;; set the niche demand randomly
    ;; set niche-demand n-values knowledge [random 2]

    ;; sets the niche demand as a full specification of what would be desirable, or all ones
    ;;set niche-demand n-values knowledge [1]

    ;; this code creates a market demand DNA string. Two options can be choosen. the first one, market fully discovered
    ;; creates an all ones market DNA. It is good to assess how well the entities will discover what a stable market wants.
    ;; It may not be suitable for those experiments where the market is supposed to change, since all knowledge is already discovered
    ;; in terms of market demand.
    ;; The second option leaves half the DNA string blank, for those simulations where the market may come to desire new discoveries through
    ;; mutations on the niche demand, where it will desire some new knowlwdgw and cease to desire some old knowledge
    ifelse market_fully_discovered? [
      set niche-demand n-values knowledge [1]
    ][
      set niche-demand n-values knowledge [[ i ] -> ifelse-value ( i < ( knowledge / 2 )) [ 0 ][ 1 ] ]
    ]

    hide-turtle
    show niche-demand
  ]
end

to mutate-market
  ask niches [
    set niche-demand n-values knowledge [random 2]
    show niche-demand
  ]
end

to market-mutation

  set market-mutation-countdown market-mutation-countdown + 1

  if market-mutation-countdown = market_mutation_period [
    mutate-market
    set market-mutation-countdown 0
  ]

end

;;*** other functions to implement
;; niche swap
;; not necessary anymore since the simulation only covers the mainstream at this iteration

;; niche learning from introduced products (crossover with consumers)
;; makes the niche call a crossover to one of the consumers. Not necessarily the fittest


;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; GUI procedures ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;; sets the shape of the entities according to their role
to select-shape

    ;; star for generators
    if generator? and not consumer? and not diffuser? and not integrator? [set shape "star"]
    ;; square for consumers
    if not generator? and consumer? and not diffuser? and not integrator? [set shape "square"]
    ;; triangle for diffusers
    if not generator? and not consumer? and diffuser? and not integrator? [set shape "triangle"]
    ;; pentagon for integrators
    if not generator? and not consumer? and not diffuser? and integrator? [set shape "pentagon"]
    ;; circle remains for hybrids, as it is the default shape

end

;; also assigns a color to the entity given its absolute fitness (an option would be to code this to evaluate if it is earning enough to live or not)
to select-fitness-color

  if color_update_rule = "fitness" [
    ;; implements the color updating by absolute fitness
    ifelse (fitness / knowledge) > 0.67 [
      set color green
    ]
    [
      ifelse (fitness / knowledge) > 0.33 [
        set color yellow
      ]
      [
        set color red
      ]
    ]
  ]

  ;; implements the color updating by survivability, the amount of iterations the entity would
  ;; be able to survive without receiving any resources
  ;; of course, it can live longer if it keeps gathering resources from the environment
  if color_update_rule = "survivability"[
    ifelse (resources > ((minimum_resources_to_live + resources * expense_to_live_growth)) * 10) [
      set color green
    ][
      ifelse (resources > ((minimum_resources_to_live + resources * expense_to_live_growth)) * 5) [
        set color yellow
      ][
        set color red
      ]
    ]

    ;; updates the color of those entities that do not have costs charged
    ;; meaning that their survival does not depend on their fitness or
    ;; on their activities
    if not consumer? and non_economical_entities? [
      set color gray
    ]
  ]

end

;; sets the size of the entity proportional to its resources, related to the amount of periods it could live without receiving resources
to set-size-entity

  set size resources / (minimum_resources_to_live + (resources * expense_to_live_growth))
  if size < 0.5 [
    set size 0.5
  ]

end

to update-link-appearance [bits1 bits2 color-link]
  ;; Evaluates whether the crossover and the mutation actually changed bits through a hamming distance
  ;; if it did, it changes the color of the link to blue and its thickness to be proportional to the number of bits changed.
  ;; If not, it colors the link red

  let knowledge-change hamming-distance bits1 bits2
  ifelse knowledge-change > 0 [
    ask my-links [
      set color color-link
      set thickness knowledge-change / knowledge
    ]
  ][
    ask my-links [
      set color red
    ]
  ]

end

to update-link-appearance-dual [ older-tech-knowledge newer-tech-knowledge  older-science-knowledge newer-science-knowledge color-link]
  ;; Evaluates whether the crossover and the mutation actually changed bits through a hamming distance
  ;; if it did, it changes the color of the link to blue and its thickness to be proportional to the number of bits changed.
  ;; If not, it colors the link red

  let new-knowledge 0
  let old-knowledge 0

  set new-knowledge sentence newer-science-knowledge newer-tech-knowledge
  set old-knowledge sentence older-science-knowledge older-tech-knowledge

  let knowledge-change hamming-distance new-knowledge old-knowledge
  ifelse knowledge-change > 0 [
    ask my-links [
      set color color-link
      ;; knowledge is multiplied by two to compensate the longer string, which includes both science and tech DNAs
      set thickness knowledge-change / ( 2 * knowledge )

      let science-change hamming-distance older-science-knowledge newer-science-knowledge
      let tech-change hamming-distance older-tech-knowledge newer-tech-knowledge

      ifelse science-change > 0 and tech-change > 0 [
        ;; if there are both kinds of knowledge change
        ;; the link shape will be the default
        ;; the color will be the one commanded by the calling procedure
      ][
        ifelse science-change > 0 [
          ;; if there is only change in the science DNA
          ;; the color will be green and the link will be traced
          set shape "traced"
          set color green
        ][
          ;; if there is only change in the tech DNA
          ;; the color will be blue and the link will be traced
          set shape "traced"
          set color blue
        ]
      ]
    ]
  ][
    ;; if there is no learning whatsoever, the link is colored red
    ask my-links [
      set color red
    ]
  ]

end

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;; other procedures ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

to define-seed

  ;; Makes the seed that will create the random numbers in the model known, making it repeatable
  ;; The seed may be choosen by the user, or randomly chosen by the model
  ;; The seed being used will be displayed in the interface in the my-seed-repeat input.
  ifelse repeat_simulation? [

    ;; Takes the seed stored in the my-seed-repeat from the last simulation / user intervention during simulation
    random-seed my-seed-repeat

  ][
    ifelse set_input_seed? [

      ;; Use a seed entered by the user
      let suitable-seed? false
      while [not suitable-seed?] [

        set my-seed user-input "Enter a random seed (an integer):"

        ;; Tries to set my-seed from the input. If it is not possible, does nothing
        carefully [ set my-seed read-from-string my-seed ] [ ]

        ;; Tests the value from my-seed. If it is suitable (number and integer), sets the random-seed
        ;; If not, asks for a new one
        ifelse is-number? my-seed and round my-seed = my-seed [
          random-seed my-seed ;; use the new seed
          output-print word "User-entered seed: " my-seed  ;; print it out
          set my-seed-repeat my-seed
          set suitable-seed? true
        ][
          user-message "Please enter an integer."
        ]
      ]

    ][
      ;; Use a seed created by the NEW-SEED reporter
      set my-seed new-seed            ;; generate a new seed
      output-print word "Generated seed: " my-seed  ;; print it out
      random-seed my-seed             ;; use the new seed
      ;; Displays the new seed in the my-seed-repeat input
      set my-seed-repeat my-seed
    ]
  ]

end

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;; (modelo DNA Protein Synthesis)
;;;;;;;;;;;;;;;;;;;;;; instructions for players ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;; presents the number of the instruction being read, and sugests the press setup if none is displayed
to-report current-instruction-label
  report ifelse-value (current-instruction = 0)
    [ "press setup" ]
    [ (word current-instruction " of " length instructions) ]
end

;; goes to next instruction on the list
to next-instruction
  show-instruction current-instruction + 1
end

;; goes to previos instruction on the list
to previous-instruction
  show-instruction current-instruction - 1
end

;; prints the selected instruction
to show-instruction [ i ]
  if i >= 1 and i <= length instructions [
    set current-instruction i
    clear-output
    foreach item (current-instruction - 1) instructions output-print
  ]
end

;; instrutcions
to-report instructions
  report [
    [
     "You will be simulating an innovation"
     "ecosystem based on knowledge flows."
     "The shapes of the entities denote"
     "their role in the ecosystem:"
     "  - Generators - stars / hollow stars"
     "  - Consumers - squares / hollow squares"
     "  - Integrators - pentagons"
     "  - Diffusers - triangles / hollow triangles"
     "  - Hybrids - circles"
     "The hollow shapes are used when a super"
     "entity is created, to differentiate it from"
     "the regular randomly create ones."
    ]
    [
     "Their color denotes several information:"
     " - Blue    - randomly assigned entities"
     " - Orange  - manually assigned entities"
     " - Cyan    - startups"
     " - Magenta - super entities"
     " - Red     - Equal or less than 33% fitness"
     "           - Less than 5 iterations in resources"
     " - Yellow  - More than 33% fitness"
     "           - More than 5 iterations in resources"
     " - Green   - More than 67% fitness"
     "           - More than 10 iterations in resources"
     " - Gray    - Entities do not receive resources from"
     "the market and are not charged at each iteration either."
    ]
    [
     "The colors of entities during run time depend on the"
     "chooser color_update_rule. You can choose:"
     " - fitness:        colors by fitness"
     " - survivability:  color by amount of resources"
     " and market survivability."
     "The color of the links represents what kind of"
     "knowledge is being shared between entities."
     "- Green - scientific knowledge has been shared"
     "- Blue - tecnologic knowledge has been shared"
     "- Yellow - both scientific and tecnologic knowledges"
     "have been shared."
      "- Red - there has been a crossover but no learning"
      "ocurred."
     "If the link is dotted (blue or green) it means that"
     "entities with both knowledges interacted, but only"
     "scientific (green traced) or tecnologic (blue traced)"
     "knowledges have been shared."

     "The chooser repeat_simulation? uses the last seed"
     "used for the random number generator or not."
     " If you choose not to repeat, the chooser "
     "set_input_seed? will prompt the user for a seed or"
     "allow the model to randomly select the seed used."
     "In any case, the seed used will be displayed in "
     "the my-seed-repeat monitor."
    ]
    [
     "When you press SETUP, if you chose to "
     "input a known seed for random numbers,"
     "you will be prompted for a integer number."
     "A population of entities with parameters "
     "randomly set is created."
     "The amount of entities at each role may"
     "be randomly or manually chosen through sliders"
     "and the random_ent_creation? chooser."
     "The first color the entities display depend"
     "on how they were created."
    ]
    [
     "Their DNA's are randomly created, and their"
     "parameters are randomly set according to the"
     "mean value and standard deviation chosen,"
     "in a normal distribution fashion."
     "Scientific knowledge and technological"
     "knowledge is assigned according to the"
     "entities roles in the ecosystem."
    ]
    [
      "Choose the initial amount of resources"
      "the entities possesses by sliding the"
      "initial_resources slider"
      "Choose the size of the markets by sliding"
      "niche_resources slider"
    ]
    [
      "Choose the amount of resources that are"
      "available at a market by sliding the"
      "niche_resources"
      "Choose minimum amount of resources to live"
      "by sliding the minimum_resources_to_live"
      "slider"
      "Choose how much do the resources necessary"
      " to remain in the market grow as the entity"
      " grows by sliding the"
      "expense_to_live_growth slider "
    ]
    [
     "The chooser  color_update_rule chooses how"
     "the colors of the entities will be updated"
     "Choosing fitness the model will color the "
     "entities according to their absolute fitness,"
     " being red up to 33% fitness, yellow up to "
     "67% fitness and green up to 100% fitness"
     "Choosing survivability will update the colors"
     " given the number of iterations the entity "
     "would be able to live without receiving "
     "any resources, being red for less than 5"
     "yellow for less than 10, and green for more"
     "than 10 iterations."
     "of course it can live longer if it keeps"
     "gathering resources from the environment"
     "but would be in trouble if competition "
     "increased or if its fitness dropped."
    ]
    [
     "The stop_trigger tells the model after how"
     "many iterations it should stop, so it will"
     "be easier to compare the results of multiple"
     "runs"
     "There is a button go for infinite loop until"
     "the stop_trigger (if defined) is reached"
     "and a button go for manual single iterations"
    ]
    [
      "The motivation to learn slider will determine"
      "how likelly it is for the entity to contact"
      "other entities."
      "It's standard deviation will create a diverse"
      "population regarding this motivation"
      "The willingness to share will determine how"
      "likelly it is for the entity to reply an"
      "interaction request by another entity"
    ]
    [
      "The mutation rate alters the rate at which"
      "entities with scientific knowledge will"
      "mutate after interacting with other entities"
      "with scientific knowledge for crossover"
      "effectively creating new knowledge"
    ]


  ]
end

; Copyright 2017 José Roberto Branco Ramos Filho
; See info tab for full copyright and license.

