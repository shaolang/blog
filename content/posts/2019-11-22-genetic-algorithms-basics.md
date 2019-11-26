---
title: "Genetic Algorithms Basics"
date: 2019-11-26T10:00:00+08:00
draft: true
---

[Genetic algorithm][ga] is another branch of artificial intelligence
that takes inspirations from Darwin's theory of evolution. Genetic algorithm
is useful in solving optimization problems where a good-enough solution
is acceptable as opposed to best solution. Good-enough because getting
the best solution can take a long time or a lot of resources to compute.
For example, genetic algorithm is useful for solving
[timetable scheduling][timetable] and [bin packing][bin-packing] problems.
Not to be confused with the similarly named [Genetic programming][gp],
genetic algorithm uses lists/arrays extensively; genetic programming uses
trees instead.

# Building blocks
Candidate solutions are encoded as chromosomes (also known as individuals);
each chromosome is made up of multiple genes that encode the solution-space.
Genetic algorithm "evolves" a population of individuals by crossing over and
mutating them towards an acceptable solution, much like the natural,
biological world. The pseudocode for a basic algorithm is as follows:

```
population = create_random_population
while continue_evolution?(population)
  population_crossed = crossover(population)
  population_mutated = mutate(population_crossed)
  evaluate(population_mutated)
  population = population_mutated
end
```

The pseudocode shows two genetic operators in use: crossover and
mutation. Crossover is the process of producing a child solution from two
or more parent solutions (think breeding/mating). By combining portions of good
solutions, the algorithm is likely able to produce better and better
child solutions as generations pass (genetic-algorithm-speak for iterations).
Mutation randomly changes some genes within the chromosome, thus creating
genetic diversity in the population. However, when crossover and mutation
rates are too high, fitter individuals might be lost during evolution; when
rates are too low, potentially strong individuals might never enter the
gene pool. Introducing elitism mitigates some effects of crossover and
mutations. Elitism keeps a handful of the fittest individuals in the
current generation untouched for interacting with individuals in the
next generation.

## Crossover
[Crossover][crossover], also known as recombination, can be implemented in
several ways:

1. Uniform crossover: each gene in selected parents has equal probability in
   being inherited by the offspring, e.g., when there are two parents, each
   gene in the offspring has a 50/50 chance of coming from either parents.
2. Single-point crossover: a common point on both parents is picked randomly
   and the offspring will inherit the first parent's genes up till that point
   and the second's genes from that point onwards as its genes, e.g., if
   the chromosome length is 10, and the common point randomly chosen is 7,
   the offspring will inherit the first 7 genes from the first parent
   and the last 3 genes from the second.
3. Two-point crossover: two common points on both parents are picked
   randomly and the offspring will inherit the genes within the two points
   from the first parent and the rest from the second, e.g., if the
   chromosome length is 10, and the common points are 3 and 6, the offspring
   will inherit the first 2 genes from the second parent, the next 4 genes
   from the first, and the remainder from the second parent.

## Mutation
[Mutation][mutation] can be implemented in one of the following ways:

1. Flip bit: invert the gene's value, if it is encoded as ones and zeroes
2. Boundary: replace the gene with a random value between the specified
   upper and lower bound
3. Swap: swap the gene selected for mutation with another in the
   same chromosome; this implementation is useful when the chromosome
   cannot allow duplicate values/genes
4. Uniform: create a new (valid) individual with randomly initialized
   chromosome, and mutate (randomly) selected genes of the individual (not
   the newly created one) by taking genes from the newly created individual,
   e.g., if genes 1, 3, and 5 from an individual (ID1) are to be mutated,
   uniform mutation will take genes from 1, 3, and 5 from the newly created
   individual and substitute them at genes 1, 3, and 5 of the mutated ID1;
   it feels a little uniform crossover, except that the offspring will
   take genes from the randomly-created individual when `mutation_rate > rand`


## Genes and chromosomes
Genes can be encoded in any way that makes sense for the problem on hand,
although they commonly are encoded numerically. For example, a binary-encoded
chromosome could be created as follows:

{{< highlight ruby "linenos=table" >}}
chromosome_length = 100
chromosome = chromosome_length.times.map { |_| rand 2 }
{{</highlight>}}

The Ruby code above creates an array of length 100 with randomly generated
numbers between 0s and 1s.

Based on the chromosome, the genetic algorithm will calculate the fitness
of the individual, usually normalized to values between 0 and 1.
The higher the fittness score, the better the individual
is/the closer the program is to solving the problem. Survival of the fittest,
if you'd like.[^1]

## Selection
[Selection][selection] is the process of choosing individuals from the
population for crossover. Although elites are favored, they shouldn't be
the only ones make available; sometimes, genes of two seemingly weak
individuals can create surprisingly fit offsprings.

Selection can be implemented in one of the following ways:

1. Fitness proportionate: individuals with higher fitness scores are more
   likely to be selected, e.g., a population of 4 individuals with fitness
   scores as `[1, 2, 3, 4, 5]`, the first individual will only have 1/15 of
   a chance in getting chosen, but the last will have a 1/3 chance. This
   method is also known as roulette wheel selection.
2. Tournament: _k_ individuals are selected randomly from the population,
   and they compete with one another in several "tournaments;" in
   deterministic tournament, the best/fittest among the selected is always
   chosen.


# First toy example: All-ones
This toy example is taken from the book
[Genetic Algorithms in Java Basics][gajb], but implemented in Ruby.[^2]
This example shows how the basic blocks can be put together to solve
problems with genetic algorithm; its goal is to evolve the individuals and
stop when one of them whose chromosome contains genes that are all ones.
Such trivial example do not require any other support classes/functions
to work; those support classes/functions will distract the goal of
understanding genetic algorithm implementations unnecessarily.

{{< highlight ruby "linenos=table" >}}
class Individual
  def Individual.of_length(length)
    Individual.new(length.times.map { |_| rand 2 })
  end

  attr_reader :chromosome, :fitness


  def initialize(chromosome)
    @chromosome = chromosome
  end


  def calc_fitness!
    @fitness = @chromosome.sum / @chromosome.size.to_f
  end


  def mate(spouse)
    offspring = @chromosome.zip(spouse.chromosome).   # pair up parents' genes
      map { |genes| genes[rand 2] }                   # randomly choose either one
    Individual.new offspring
  end


  def mutate(mutation_rate)
    mutated = @chromosome.map do |gene|
      # when mutation should happen, then
      #   when gene = 1, (1 - 1).abs => 0; when 0, (0 - 1).abs => 1
      mutation_rate > rand ? (gene - 1).abs : gene
    end

    Individual.new mutated
  end


  def <=>(other)
    -1 * (@fitness <=> other.fitness)
  end


  def to_s
    "#{@chromosome.join ''} (fitness: #{@fitness})"
  end
end
{{</highlight>}}

`Individual` largely keeps track of the chromosome and its fitness. Specific
to genetic algorithms are `calc_fitness`, `mate`, and `mutate` functions.
`calc_fitness` simply 1 divide the sum of the chromosome, so nothing really
to see here.

`mate` is the method used in crossovers; it accepts another `Individual`
instance:

1. Pairs up the parents' genes (e.g., if parents' genes were `[1, 2]`, and
   `[3, 4]`, `[1, 2].zip [3, 4]` returns `[[1, 3], [2, 4]]`
2. Chooses randomly from the pairs of genes in the block given to `map`
3. Creates and returns the offspring with the new genes

`mutate` is given the mutation rate, and it simply run through the entire
chromosome and flip bit the gene when `mutation_rate > rand`.

`<=>` overrides the comparison to allow sorting fitnesses in descending order.

{{< highlight ruby "linenos=table, linenostart=47" >}}
class Population
  def Population.of_size(size, chromosome_length)
    Population.new(size.times.map { |_| Individual.of_length chromosome_length})
  end


  def initialize(individuals)
    @individuals = individuals
  end


  def eval_individuals!
    @individuals.each { |i| i.calc_fitness! }
    @individuals.sort!
    @population_fitness = @individuals.map { |i| i.fitness }.sum
  end


  def crossover_individuals(crossover_rate, elitism_count)
    elites = @individuals.take elitism_count                      # keep elites for next gen
    crossed = @individuals[elitism_count..].map do |individual|   # cross the rest
      if crossover_rate > rand
        spouse = select_parent    # helper method
        individual.mate spouse
      else
        individual
      end
    end

    Population.new(elites + crossed)
  end


  def mutate_individuals(mutation_rate, elitism_count)
    elites = @individuals.take elitism_count
    mutated = @individuals[elitism_count..].map do |individual|
      individual.mutate mutation_rate
    end

    Population.new(elites + mutated)
  end


  def fittest
    @individuals[0]
  end


  private
  def select_parent
    roulette_wheel_pos = rand * @population_fitness
    spin_wheel_pos = 0

    for individual in @individuals
      spin_wheel_pos += individual.fitness

      return individual if spin_wheel_pos >= roulette_wheel_pos
    end

    @individuals[-1]  # return last individual if for-loop completes its run
  end
end
{{</highlight>}}

Population is the "container" for individuals, that has a class method
to create a population with randomly initialized individuals (delegating to
`Individual.of_length`). Both `crossover_individuals` and `mutate_individuals`
methods are given the `elitism_count` parameter, so as to ensure the
preservation of elites for the next generation (refer back to
[Building blocks](#building-blocks) for info).

`crossover_individuals` calls the private `select_parent` when it needs
to select a parent for crossover with the current individual it is looping
over. `select_parent` uses the fitness proportionate selection method to
select the parent. Notice that the individuals available for selection
include the elites (as evident on line 100 `for individual in @individuals`
in `select_parent` method).

`mutate_individuals` simply loops through `@individuals` and delegates
the mutation to the individual's `mutate` method.

`eval_individuals` makes all individuals to calculate their own fitness,
and it also calculates the population's overall fitness (tracked by
`@population_fitness` instance variable). Normally, there isn't a
need to keep track of the population's fitness; the need arises because
fitness proportionate selection method used in `select_parent` needs
to know the population's overall fitness to function.


{{<highlight ruby "linenos=table, linenostart=110">}}
if __FILE__ == $PROGRAM_NAME
  mutation_rate = 0.01
  crossover_rate = 0.95
  elitism_count = 3
  population_size = 100
  chromosome_length = 50

  population = Population.of_size population_size, chromosome_length
  population.eval_individuals!

  curr_generation = 1

  while population.fittest.fitness < 1.0
    puts "Generation #{curr_generation} fittest: #{population.fittest}"

    next_gen = population.crossover_individuals crossover_rate, elitism_count
    next_gen = next_gen.mutate_individuals mutation_rate, elitism_count
    next_gen.eval_individuals!

    population = next_gen
    curr_generation += 1
  end

  puts "Solution found in generation #{curr_generation}"
  puts population.fittest
end
{{</highlight>}}

The main program ties both `Population` and `Individual` up by implementing
the pseudocode as outlined in [Building blocks](#building-blocks) (except
the addition of outputting the current fittest in the loop, and the
fittest at the end of the program). With the given values to
`mutation_rate`, `crossover_rate` and `elisitm_count`, the program should
arrive at the solution within 400 to 600 generations most of the time.

# Real-world example: Class scheduling
Another example borrowed from the book but with a much simpler implementation:
the algorithm attempts to generate a class schedule that meets the following
hard criteria:

1. Room must be able to fit in all students from the student group (assuming
   full-attendance)
2. No room is double-booked for each timeslot
3. No professor is teaching in any two sessions at the same time

Soft criteria such as ensuring the professor should not teach more than 2
consecutive sessions a day are not covered to simplify the discussion. By
not catering for soft criteria, the algorithm does not account for the
quality of the solutions found.

{{< highlight ruby "linenos=table,linenostart=1" >}}
ROOMS = {1 => {name: 'A1', capacity: 15},
         2 => {name: 'B1', capacity: 30},
         4 => {name: 'D1', capacity: 20},
         5 => {name: 'F1', capacity: 25}}

SLOTS = {1  => 'Mon 09:00 - 11:00',
         2  => 'Mon 11:00 - 13:00',
         3  => 'Mon 13:00 - 15:00',
         4  => 'Tue 09:00 - 11:00',
         5  => 'Tue 11:00 - 13:00',
         6  => 'Tue 13:00 - 15:00',
         7  => 'Wed 09:00 - 11:00',
         8  => 'Wed 11:00 - 13:00',
         9  => 'Wed 13:00 - 15:00',
         10 => 'Thu 09:00 - 11:00',
         11 => 'Thu 11:00 - 13:00',
         12 => 'Thu 13:00 - 15:00',
         13 => 'Fri 09:00 - 11:00',
         14 => 'Fri 11:00 - 13:00',
         15 => 'Fri 13:00 - 15:00'}

PROFESSORS = {1 => 'Dr P Smith',
              2 => 'Mrs E Mitchell',
              3 => 'Dr R Williams',
              4 => 'Mr A Thompson'}

MODULES = {1 => {name: 'Computer Science',  professors: [1, 2]},
           2 => {name: 'English',           professors: [1, 3]},
           3 => {name: 'Maths',             professors: [1, 2]},
           4 => {name: 'Physics',           professors: [3, 4]},
           5 => {name: 'History',           professors: [4]},
           6 => {name: 'Drama',             professors: [1, 4]}}

STUDENT_GROUPS = {1  => {size: 10, modules: [1, 3, 4]},
                  2  => {size: 30, modules: [2, 3, 5, 6]},
                  3  => {size: 18, modules: [3, 4, 5]},
                  4  => {size: 25, modules: [1, 4]},
                  5  => {size: 20, modules: [2, 3, 5]},
                  6  => {size: 22, modules: [1, 4, 5]},
                  7  => {size: 16, modules: [1, 3]},
                  8  => {size: 18, modules: [2, 6]},
                  9  => {size: 24, modules: [1, 6]},
                  10 => {size: 25, modules: [3, 4]}}

SESSIONS = STUDENT_GROUPS.map do |kv|
  sg_id = kv[0]
  size = kv[1][:size]
  kv[1][:modules].map { |m| {sg_id: sg_id, size: size, module: m} }
end.reduce(:+)
{{</ highlight >}}

To keep things really simple, the data are hard-coded as maps. The last line
calculates the total number of sessions the algorithm needs to work out.
`professors` key in `MODULES` lists the professors (by ID) that can
teach that module; similarly, `modules` key in `STUDENT_GROUPS` lists
the modules that group is taking.

`SESSIONS` translates `STUDENT_GROUPS` into (class) sessions that needed to
be conducted, e.g., the algorithm needs to schedule 3 sessions for student
group 1: one for Computer Science, one for Maths, and one for Physics.

{{< highlight ruby "linenos=table,linenostart=52" >}}
def random_room
  ROOMS.keys[rand ROOMS.size]
end


def random_slot
  SLOTS.keys[rand SLOTS.size]
end


def random_professor(sess_id)
  mod_id = SESSIONS[sess_id][:module]
  professors = MODULES[mod_id][:professors]
  professors[rand professors.size]
end
{{</ highlight >}}

The above are helper functions for randomly choosing timeslot, and professor.
When choosing a professor randomly, the function needs to know for which
module's list of professors it need to look at.

{{< highlight ruby "linenos=table,linenostart=69" >}}
class Individual
  GENE_GROUP_SIZE = 3
  ROOM_IX, SLOT_IX, PROF_IX = GENE_GROUP_SIZE.times.to_a

  def Individual.random_schedule
    Individual.new(SESSIONS.size.times.map do |sess_id|
      mod_id = SESSIONS[sess_id][:module]
      [random_room, random_slot, random_professor(mod_id)]
    end.reduce(:+))
  end


  attr_reader :chromosome, :fitness


  def initialize(chromosome)
    @chromosome = chromosome
  end


  def calc_fitness
    gene_groups = to_gene_groups(@chromosome)
    clashes = from_room_capacity_vs_group_size gene_groups
    clashes += from_room_availability gene_groups
    clashes += from_professor_availability gene_groups

    @fitness = 1.0 / (clashes + 1)
  end


  def mate(spouse)
    offspring = @chromosome.zip(spouse.chromosome).map do |genes|
      genes[rand 2]
    end

    Individual.new offspring
  end


  def mutate(mutation_rate)
    other = Individual.random_schedule
    mutated = @chromosome.zip(other.chromosome).map do |genes|
      genes[mutation_rate > rand ? 1 : 0]
    end

    Individual.new mutated
  end


  def to_s
    schedule = to_gene_groups(@chromosome).map do |v|
      sess, i = v
      room = "Room #{ROOMS[sess[ROOM_IX]][:name]}"
      slot = SLOTS[sess[SLOT_IX]]
      professor = PROFESSORS[sess[PROF_IX]]
      mod = MODULES[SESSIONS[i][:module]][:name]
      sg_id = SESSIONS[i][:sg_id]

      [sg_id, slot, mod, room, professor]
    end.sort.map do |s|
      s.join ','
    end.join "\n"

    "Schedule fitness #{@fitness}\n#{schedule}"
  end


  def <=>(other)
    -1 * (@fitness <=> other.fitness)
  end


  private
  def to_gene_groups(chromosome)
    chromosome.each_slice(GENE_GROUP_SIZE).zip SESSIONS.size.times.to_a
  end


  def from_room_capacity_vs_group_size(gene_groups)
    gene_groups.map do |gg|
      sess, i = gg
      room_capacity = ROOMS[sess[ROOM_IX]][:capacity]
      group_size = SESSIONS[i][:size]
      room_capacity < group_size ? 1: 0
    end.sum
  end


  def from_room_availability(gene_groups)
    gene_groups.map do |gg|
      target = [gg[ROOM_IX], gg[SLOT_IX]]

      # must divide by 2, 'cos the following will always count itself once
      gene_groups.map { |x| [x[ROOM_IX], x[SLOT_IX]] == target ? 1 : 0 }.sum / 2
    end.sum
  end


  def from_professor_availability(gene_groups)
    gene_groups.map do |gg|
      target = [gg[PROF_IX], gg[SLOT_IX]]

      # must divide by 2, 'cos the following will always count itself once
      gene_groups.map { |x| [x[PROF_IX], x[SLOT_IX]] == target ? 1 : 0 }.sum / 2
    end.sum
  end
end
{{</ highlight >}}

Woah! This `Individual` implementation is huge! Let's break that down.
The three biggest changes to `Individual` are how it creates a random
individual, how it calculates its fitness, and how it mutates. Or put
another way, only its `mate` method is identical to the toy example's.

Unlike the toy example, the chromosome cannot be examined in isolation:
the entire chromosome needs to be examined in groups of 3 (evident at
lines 70 and 71) where:

1. First gene in group represents the room ID
2. Second gene represents the timeslot ID
3. Third represents the professor ID

Each group corresponds to the (class) session in `SESSIONS`. For simplicity
in discussion, we'll call each group a gene group.

`Individual.random_schedule` uses the helper functions to generate
the chromosome, and those helper functions ensure that the genes are
always correct. `Individual.random_schedule` can't just do a `rand(n) + 1`
for each gene, as IDs in the hard-coded data aren't always in running order,
e.g., there is no room ID 3. Such rigorous control ensures that all
chromosomes are valid (even if they contain clashes).

Because of the need to ensure the chromosomes are valid, `mutate` uses uniform
mutation (refer to description covered in [mutation](#mutation)); it creates
a valid random individual and uses that individual's genes at place where
mutation should occur.

`calc_fitness` (seemingly) has the most complicated change: by examining
all gene groups in the chromosome, it determines the number of clashes (or
if you'd like, breaks in hard constraints) by calling the private helper
methods `from_room_capacity_vs_group_size`, `from_room_availability`, and
`from_professor_availability`. As highlighted in the code comments on lines
165 and 174, clashes counted in the latter two methods must be divided by two,
as the counts include the (target) session itself. The code makes use of
Ruby's integral division round-down behavior: Ruby returns `0` for the
expression `1 / 2`, `1` for `2 / 2` and `3 / 2`. When there are 2 or more
counts of the same constraint, we don't really care whether there are 2 or 3 or
more breaks, thus Ruby's round-down behavior is good enough to account for the
breaks.

{{< highlight ruby "linenos=table,linenostart=178" >}}
class Population
  def Population.random_schedules(pop_size)
    Population.new(pop_size.times.map { |_| Individual.random_schedule })
  end


  def initialize(individuals)
    @individuals = individuals
  end


  def eval_individuals!
    @individuals.each { |i| i.calc_fitness }
    @individuals = @individuals.sort
  end


  def crossover_individuals!(crossover_rate, elitism_count, tournament_size)
    elites = @individuals.take elitism_count
    crossed = @individuals[elitism_count..].map do |i|
      if crossover_rate > rand
        i.mate select_parent(tournament_size)
      else
        i
      end
    end

    Population.new(elites + crossed)
  end


  def mutate_individuals!(mutation_rate, elitism_count)
    elites = @individuals.take elitism_count
    mutated = @individuals[elitism_count..].map { |i| i.mutate mutation_rate }

    Population.new(elites + mutated)
  end


  def fittest
    @individuals[0]
  end


  private
  def select_parent(tournament_size)
    candidates = @individuals.shuffle.take tournament_size
    candidates.sort[0]
  end
end
{{</ highlight >}}

The main difference in `Population` is `select_parent` employs deterministic
tournament selection (refer to description in [Selection](#selection)).

{{< highlight ruby "linenos=table,linenostart=230" >}}
if __FILE__ == $PROGRAM_NAME
  pop_size = 100
  mutation_rate = 0.01
  crossover_rate = 0.9
  elitism_count = 2
  tournament_size = 5
  max_generations = 1000
  curr_gen = 1

  population = Population.random_schedules pop_size
  population.eval_individuals!


  while curr_gen < max_generations && population.fittest.fitness < 1.0
    puts "Generation #{curr_gen} fittest: #{population.fittest.fitness}"

    next_gen = population.crossover_individuals!(crossover_rate, elitism_count,
                                                 tournament_size)
    population = next_gen.mutate_individuals! mutation_rate, elitism_count
    population.eval_individuals!

    curr_gen += 1
  end

  puts "Best solution found: #{population.fittest}"
end
{{</ highlight >}}

The main program is largely the same, except the introduction of
`tournament_size` that is used in selecting parents for crossovers. The
following shows one possibility of the schedule (reformatted for the web):

Student Group | Timeslot          | Module          | Room | Professor
:------------:|:-----------------:|:---------------:|------|----------
1             | Fri 09:00 - 11:00 |Maths            |Room B1|Dr R Williams
1             | Mon 13:00 - 15:00 |Physics          |Room D1|Mrs E Mitchell
1             | Tue 09:00 - 11:00 |Computer Science |Room D1|Dr P Smith
2             | Fri 09:00 - 11:00 |History          |Room B1|Mr A Thompson
2             | Mon 11:00 - 13:00 |English          |Room B1|Mr A Thompson
2             | Mon 13:00 - 15:00 |Drama            |Room B1|Mr A Thompson
2             | Wed 11:00 - 13:00 |Maths            |Room B1|Dr P Smith
3             | Mon 13:00 - 15:00 |History          |Room D1|Mr A Thompson
3             | Tue 13:00 - 15:00 |Maths            |Room B1|Dr P Smith
3             | Wed 09:00 - 11:00 |Physics          |Room D1|Dr P Smith
4             | Tue 13:00 - 15:00 |Computer Science |Room B1|Dr P Smith
4             | Wed 09:00 - 11:00 |Physics          |Room F1|Dr P Smith
5             | Thu 11:00 - 13:00 |History          |Room D1|Mr A Thompson
5             | Thu 11:00 - 13:00 |Maths            |Room B1|Dr P Smith
5             | Wed 11:00 - 13:00 |English          |Room D1|Mr A Thompson
6             | Thu 11:00 - 13:00 |Computer Science |Room B1|Mrs E Mitchell
6             | Tue 09:00 - 11:00 |History          |Room F1|Mr A Thompson
6             | Tue 09:00 - 11:00 |Physics          |Room B1|Mrs E Mitchell
7             | Fri 13:00 - 15:00 |Maths            |Room D1|Dr P Smith
7             | Tue 09:00 - 11:00 |Computer Science |Room D1|Dr P Smith
8             | Thu 13:00 - 15:00 |English          |Room F1|Mr A Thompson
8             | Tue 09:00 - 11:00 |Drama            |Room D1|Mr A Thompson
9             | Mon 13:00 - 15:00 |Drama            |Room B1|Mr A Thompson
9             | Tue 13:00 - 15:00 |Computer Science |Room F1|Mrs E Mitchell
10            | Fri 09:00 - 11:00 |Maths            |Room F1|Dr R Williams
10            | Thu 09:00 - 11:00 |Physics          |Room B1|Mrs E Mitchell

As mentioned, soft criteria are not accounted for by this algorithm, but
implementing requires a slight change in calculating the individual's
fitness:

1. Breaks in hard criteria attract large negative scores, e.g., -100
   for each break
2. Breaks in soft criteria attract small positive scores, e.g., +1
   for each break

Care must be taken for such scoring method to ensure points from soft
criteria do not cancel out points from hard ones, e.g., 1 hard break
(-100 points) is not cancelled out by 100 soft breaks (+100 points).
When soft criteria are included, the fitness score may not necessarily be 1
because the (fittest) solution may not meet all soft criteria, but solutions
that consider soft criteria definitely differ from one other in terms of
quality: a solution with fitness 0.5 (one soft break) is better than another
with fitness 0.33 (two soft breaks).


# Last note on crossovers
As the two examples above use uniform crossover, let's touch a bit on
single-point crossover. What follows is equally applicable to two-point
crossover.

The algorithm cannot simply choose any point in the chromosome for
single-point crossover, as that may break the chromosome's validity.
As seen in the class scheduling example, genes should be viewed in groups
of three. Therefore, the following implementation is wrong:

{{<highlight ruby "linenos=table,hl_lines=2">}}
def mate(spouse)
  common_point = rand @chromosome.size    # NO!
  offspring = @chromosome.take(common_point) + spouse.chromosome[common_point..]
  Individual.new offspring
end
{{</highlight>}}

Assuming `rand @chromosome.size` returns 1, this implementation of `mate`
will likely invalidate the first three genes that should be viewed as a group.

The following shows the correct implementation that crosses over the genes
in gene groups:

{{<highlight ruby "linenos=table,hl_lines=2-3">}}
def mate(spouse)
  groups = @chromosome.size / GENE_GROUP_SIZE     # assuming GENE_GROUP_SIZE = 3
  common_point = rand(groups) * GENE_GROUP_SIZE
  offspring = @chromosome.take(common_point) + spouse.chromosome[common_point..]
  Individual.new offspring
end
{{</highlight>}}

# Closing thoughts
Genetic algorithms aren't that difficult, considering that the class
schedule example has only 255 LOC. Adding optimizations such as
fitness calculation [memoization][memoization], [adaptive algorithms][^3]
will probably add another 10 - 100 LOC, but even so, it's doing so much
with so little code.

While genetic algorithm isn't as "sexy" as machine learning, it's definitely
one useful tool for solving optimization problems.

[^1]: Herbert Spencer coined this [phrase][sotf] after reading Charles
      Darwin's _On the Origin of Species_. This phrase is best understood
      as "survival of the form that will leave the most copies of itself
      in successive generations." On a side note, the other oft-heard
      phrase "[i]t is not the strongest of the species that survives, nor the
      most intelligent, but the one most adaptable to change" did not originate
      from Charles Darwin. A Louisiana State University business professor,
      Leon C. Megginson, [first used that phrase][strongest]--his idiosyncratic
      interpretion on Darwin's _On the Origin of Species_--at a convention
      in 1963.
[^2]: Because the overly verbose Java syntax masks the otherwise simple
      concepts of genetic algorithm. Other than the differenece in language
      choice, the following also reorganizes the code to make it more
      cohesive and simpler to understand (the book's insistence in create a
      specific `GeneticAlgorithm` class hurts cohesiveness big time).
[^3]: Adaptive genetic algorithm tunes various parameters (mutation rate,
      crossover rate, etc) based on the state of the algorithm, e.g., it
      increases the mutation rate to encourage increasing the gene pool
      divserity when it detects very little difference between the individuals.

[ga]: https://en.wikipedia.org/wiki/Genetic_algorithm
[gp]: https://en.wikipedia.org/wiki/Genetic_programming
[timetable]: https://medium.com/@vijinimallawaarachchi/time-table-scheduling-2207ca593b4d
[bin-packing]: https://www.researchgate.net/publication/273121476_A_genetic_algorithm_for_the_three-dimensional_bin_packing_problem_with_heterogeneous_bins
[sotf]: https://en.wikipedia.org/wiki/Survival_of_the_fittest
[strongest]: https://quoteinvestigator.com/2014/05/04/adapt
[crossover]: https://en.wikipedia.org/wiki/Crossover_(genetic_algorithm)
[mutation]: https://en.wikipedia.org/wiki/Mutation_(genetic_algorithm)
[selection]: https://en.wikipedia.org/wiki/Selection_(genetic_algorithm)
[gajb]: https://www.apress.com/us/book/9781484203293
[memoization]: https://en.wikipedia.org/wiki/Memoization
